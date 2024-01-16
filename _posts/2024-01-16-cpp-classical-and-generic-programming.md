---
layout: posts
title: "단형성, 다형성, 그리고 제너릭 프로그래밍"
categories:
  - Programming
tags:
  - cpp
---

## 서론

프로그래밍을 하다 보면, 비슷하거나 같은 동작을 하는 함수들을 묶어서 재활용해야 한다.
이때 `c++`에서는 상속을 사용하거나 템플릿을 사용한다.
단순하게 코드를 작성한다면 중복되는 코드들이 불필요하게 생길 것이다.

어느 경우에는 하나의 클래스에만 적용되는 메서드를 원할 수도 있고
어느 경우에는 연관성을 지닌 여러 클래스에만 적용되는 메서드를 원할 수 있다.

연관성이 상속이라면, 부모 클래스가 갖는 공통 메서드를 이용한다.
연관성이 템플릿이라면, `템플릿에 필요한 조건들의 집합 이름`을 `concept`이라 하는데,
같은 `concept`을 만족하는 클래스들에 동일한 동작을 적용할 수 있다.

* **상속**
  1. 반드시 `virtual tables`에 반복적으로 질의하여 `virtual methods`를 호출하기 때문에
  런타임에만 실행되고 오버헤드가 발생한다.
  1. 처음 결정한 메서드 시그니처를 바꾸기 힘들다. 
  예를 들어, `void AttackMonster()`가 나중에 공격에 성공했는 지의 여부(`bool`)를 리턴하거나
  피해를 준 데미지의 양(`float`)을 리턴하고 싶어도 그럴 수 없다. 
  1. 생성되는 바이너리 코드의 양이 제너릭 프로그래밍에 비해 적다.

* **일반화 프로그래밍**
  1. 컴파일 시간에 메서드들이 작성되어 동작을 결정하므로 모든 메서드들이 인라인 호출이 가능하며
  곧바로 호출하므로 속도가 빠르다. 
  1. 함수 시그니처의 영향을 크게 받지 않는다. 
  예를 들어, `R Add(T a, T b)`는 `int`, `float`과 같은 기본형뿐만 아니라
  `Vector`, `Damage`과 같은 다양한 타입들에 대해 비슷하거나 동일한 동작을 적용할 수 있다.


## 단형성 (Classical Monomorphism)

전통적인 방법으로 하나의 클래스만 고려했다.

~~~cpp
class ArrayOfInt
{
    int Data[5] ={1,2,3,4,5};
public:
    int Size() const { return 5; }

    int& operator[](int i) 
    { 
        return const_cast<int&>((*static_cast<const ArrayOfInt*>(this))[i]);
    }

    const int& operator[](int i) const
    { 
        assert(0 <= i && i < 5);
        return Data[i]; 
    }
};

int Sum(const ArrayOfInt& Array)
{
    int Result = 0;
    for (int i = 0; i < Array.Size(); i++)
    {
        Result += Array[i];
    }
    return Result;
}

void Multiply(ArrayOfInt& ArrayRef, int Factor)
{
    for (int i = 0; i < ArrayRef.Size(); i++)
    {
        ArrayRef[i] *= Factor;
    }
}

void Test()
{
    ArrayOfInt Array;

    auto PrintArray = [&] (const char* cstr) {
        std::cout << cstr;
        for (int i = 0; i < Array.Size(); i++) {
            std::cout << Array[i] << " ";
        }
        std::cout << std::endl;
    };

    PrintArray("Array: ");
    std::cout << "Sum: " << Sum(Array) << std::endl;
    Multiply(Array, 2);
    PrintArray("Array Multiplied: ");
}
~~~

~~~terminal
Array: 1 2 3 4 5
Sum: 15
Array Multiplied: 2 4 6 8 10
~~~

`Sum` 함수는 오직 `ArrayOfInt`의 `child types`에 대해서만 작동한다.
다른 타입을 인수로 받으면 컴파일 에러가 나타난다.
이러한 종류의 함수는 `concrete or monomorphic function` 이라고 부른다.
만약 `c++ stl`이 이와 같은 `concrete` 알고리즘만 제공했다면 특정 타입들에 대해서만 작동했을 것이다.

## 다형성 (Classical Polymorphism)

이번에는 `ArrayOfInt`와 `ListOfInt`가 모두 정수형 데이터를 보관하는 
컨테이너 역할을 한다는 점에서 부모 클래스 `ContainerOfInt`로부터 상속받는다.

~~~cpp
class ContainerOfInt
{
public: // 인터페이스 설계가 중요하다.
    virtual int Size() const = 0;
    int& operator[](int i)
    {
        return const_cast<int&>((*static_cast<const ContainerOfInt*>(this))[i]);
    }
    virtual const int& operator[](int) const = 0;
    virtual void Add(int) = 0;
};

class ArrayOfInt: public ContainerOfInt
{
    int* Datas = nullptr;
    int Capacity = 10;
    int Length = 0;
public:
    ArrayOfInt() { Datas = new int[Capacity]; }
    ~ArrayOfInt()
    {
        delete[] Datas;
        Datas = nullptr;
    }

    //~ Begin ContainerOfInt Interface.
    virtual int Size() const override { return Length; }
    using ContainerOfInt::operator[];
    virtual const int& operator[](int i) const override { return Datas[i]; }
    virtual void Add(int InData) override
    {
        if (Capacity <= Length)
        {
            const int NewCapacity = Capacity * 2;
            if (int* NewDatas = new(std::nothrow) int[NewCapacity])
            {
                for (int i = 0; i < Length; i++)
                {
                    NewDatas[i] = Datas[i];
                }
                delete[] Datas;
                Datas = NewDatas;
                Capacity = NewCapacity;
            }
        }
        Datas[Length++] = InData;
    }
    //~ End ContainerOfInt Interface.
};

class ListOfInt: public ContainerOfInt
{
    struct Node
    {
        int Data;
        Node* Next;
    };
    Node* Root = nullptr;
    Node* Last = nullptr;
    int NumNode = 0;

public:
    ~ListOfInt()
    {
        Node* Temp;
        while (Root)
        {
            Temp = Root;
            Root = Root->Next;
            delete Temp;
        }
        Temp = nullptr;
    }

    //~ Begin ContainerOfInt Interface.
    virtual int Size() const override { return NumNode; }
    using ContainerOfInt::operator[];
    virtual const int& operator[](int i) const override
    {
        assert(0 <= i && i < NumNode);
        Node* Node = Root;
        for (int Count = 0; Count < i; Count++)
        {
            Node = Node->Next;
        }
        return Node->Data;
    }
    virtual void Add(int InData) override
    {
        if (Node* NewNode = new(std::nothrow) Node{InData, nullptr})
        {
            if (Last) Last = (Last->Next = NewNode);
            else Root = Last = NewNode;
            NumNode++;
        }
    }
    //~ End ContainerOfInt Interface.
};

int Sum(const ContainerOfInt& Container)
{
    // ArrayOfInt: T(n) = Theta(n)
    // ListOfInt:  T(n) = Theta(n^2)
    int Result = 0;
    for (int i = 0; i < Container.Size(); i++)
    {
        Result += Container[i];
    }
    return Result;
}

void Multiply(ContainerOfInt& ContainerRef, int Factor)
{
    // ArrayOfInt: T(n) = Theta(n)
    // ListOfInt:  T(n) = Theta(n^2)
    for (int i = 0; i < ContainerRef.Size(); i++)
    {
        ContainerRef[i] *= Factor;
    }
}

void Test()
{
    auto Print = [] (const char* cstr, ContainerOfInt& ContainerRef){
        std::cout << cstr;
        for (int i = 0; i < ContainerRef.Size(); i++) {
            std::cout << ContainerRef[i] << " ";
        }
        std::cout << std::endl;
    };

    ArrayOfInt Array;
    ListOfInt List;
    for (int i = 1; i <= 5; i++) {
        Array.Add(i);
        List.Add(5 - i + 1);
    }

    Print("Array: ", Array);
    Print("List: ", List);

    std::cout << "Sum of Array: " << Sum(Array) << std::endl;
    std::cout << "Sum of List: " << Sum(List) << std::endl;

    Multiply(Array, 2);
    Multiply(List, -1);

    Print("Array: ", Array);
    Print("List: ", List);

    for (int i = 0; i < 5; i++) {
        Array[i] = List[i] = 0;
    }
    Print("Array: ", Array);
    Print("List: ", List);
}
~~~

> 얕은 복제를 하는 문제점이 있지만 생략했다.

~~~output
Array: 1 2 3 4 5
List: 5 4 3 2 1
Sum of Array: 15
Sum of List: 15
Array: 2 4 6 8 10
List: -5 -4 -3 -2 -1
Array: 0 0 0 0 0
List: 0 0 0 0 0
~~~

`ArrayOfInt`와 `ListOfInt`는 공통 부모 `ContainerOfInt`로부터 상속받았다.
두 클래스는 모두 정수형 데이터의 컨테이너 역할을 하기 때문에 같은 메서드를 갖는다.
`ArrayOfInt`는 가변 배열을 컨테이너로 갖고, `ListOfInt`는 연결리스트를 컨테이너로 갖는다.
상속을 하므로써 가상 메서드(`Add`, `Size`, `operator[] const`)를 활용하거나, 공통 부모 메서드(`operator[] non-const`)를 활용할 수 있다.

## 일반화 프로그래밍 (Generic Programming)

`Multiply` 메서드는 `ArrayOfInt`와 `ListOfInt` 뿐만 아니라 여러 클래스에서 재활용할 수 있다.

~~~cpp
template <class InContainerType, class InFactorType>
void Multiply(InContainerType& Container,InFactorType&& Factor)
{
    for (int i = 0; i < Container.Size(); i++)
    {
        Container[i] *= Factor;
    }
}

class DamageArray: public std::vector<float>
{
public:
    DamageArray(std::initializer_list<float> Init) : std::vector<float>(Init) {}
    int Size() const { return static_cast<int>(size()); }
};

void Test()
{
    auto Print = [] (const char* cstr, auto&& Container){
        std::cout << cstr;
        for (int i = 0; i < Container.Size(); i++) {
            std::cout << Container[i] << " ";
        }
        std::cout << std::endl;
    };

    ArrayOfInt Array;
    ListOfInt List;
    for (int i = 1; i <= 5; i++) {
        Array.Add(i);
        List.Add(5 - i + 1);
    }
    DamageArray DamagesToApply { 1.f, 2.f, 3.f, 4.f, 5.f };

    Multiply(Array, 2);
    Multiply(List, -1);
    Multiply(DamagesToApply, 10.f);

    Print("Array: ", Array);
    Print("List: ", List);
    Print("std::vector: ", DamagesToApply);
}
~~~

~~~output
Array: 2 4 6 8 10
List: -5 -4 -3 -2 -1
std::vector: 10 20 30 40 50
~~~

템플릿 함수 `Multiply`가 `InContainerType`를 사용할 수 있는 조건을
명확하게 표현할 수 있다면 좋은 프로그램 디자인일 것이다.
그러한 `a named set of requirements`을 `c++`에서는 `concept`이라 부른다.
`c++20`부터 `concept keyword`를 통해 직접적으로 공식화할 수 있다.

~~~cpp
template < template-parameter-list >
concept concept-name attr(optional) = constraint-expression;
~~~

`InContainerType`의 `concept`은 다음과 같다.
1. `Size()` 메서드를 갖고 리턴 타입이 정수 타입과 비교 가능해야 한다.
2. `operator[]` 연산자를 갖고 리턴 타입이 레퍼런스이고 `InFactorType`과 곱셈 연산이 가능해야 한다.

## 결론

단형성은 코드가 복잡하지 않아서 곧바로 이해하기 쉽다.

다형성은 데이터와 메서드가 유사한 클래스들을 구현하기 좋다. 
공통 인터페이스들과 공통 데이터들을 결정할 수 있어서 구조적이다.
그래서 인터페이스 시그니처와 네이밍 등 설계가 중요하다.
한번 결정한 인터페이스를 바꾸기 번거롭다.

일반화 프로그래밍은 서로 연관성이 없는 클래스들에 대해 같은 동작을 하는 템플릿 함수를
하나만 구현하면 되므로 효율적인 코드 작성이 가능하다.
다만, 한번에 코드를 이해하기 어렵다. 그래서
구현부에서 주로 사용하고 사용측에는 `a wrapper class`를 만들어 주는 것이 좋아보인다.

`c++20`부터 `concept`이 도입되었지만,
아직은 `Unreal Engine 5.3 Documents`의 코딩 표준에서 `concept`에 대한 지침사항이 안 적혀 있다.
추후에 공부할 예정이다. 언젠가는..
