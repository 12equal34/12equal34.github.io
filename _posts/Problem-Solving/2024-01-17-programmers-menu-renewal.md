---
title: "[C++] 2021 KAKAO BLIND RECRUITMENT 메뉴 리뉴얼"
categories:
  - Programmers
tags:
  - [Algorithm, Problem Solving]
---

## 문제

[![Problem Image]({{ site.url }}{{ site.baseurl }}/assets/images/programmers/menu-renewal.png)](https://school.programmers.co.kr/learn/courses/30/lessons/72411)

## 풀이

~~~cpp
#include <string>
#include <vector>
#include <map>
#include <algorithm>

using namespace std;

vector<string> Combination(const string& InStr, map<string,vector<string>>& Memo)
{
    if (InStr.size() == 0) return {};
    else if (InStr.size() == 1) return {string{},string(1,InStr[0])};

    const string InStrSub =InStr.substr(1);
    if (Memo.count(InStrSub) == 0) Memo[InStrSub] = Combination(InStrSub, Memo);

    vector<string> Result;
    Result.reserve(Memo[InStrSub].size() * 2);
    for (const string& SubStr : Memo[InStrSub])
    {
        Result.push_back(SubStr);
        Result.push_back(InStr[0] + SubStr);
    }
    return Result;
}


vector<string> solution(vector<string> orders, vector<int> course)
{
    vector<string> answer;
    map<string,int> MenuToCount;
    vector<vector<string>> CountToMenus(21);

    auto Comp = [](const string& a, const string& b) {
        if (a.size() == b.size()) return a < b;
        return a.size() < b.size();
    };

    for (string& Order : orders)
    {
        sort(Order.begin(), Order.end());

        map<string,vector<string>> CombinationMemo;
        for (const string& Menu : Combination(Order, CombinationMemo))
        {
            if (Menu.size() < 2) continue;

            const int PrevCount = MenuToCount[Menu]++;
            const int CurrentCount = PrevCount + 1;

            if (PrevCount > 0)
            {
                auto It = remove(CountToMenus[PrevCount].begin(), CountToMenus[PrevCount].end(), Menu);
                CountToMenus[PrevCount].erase(It, CountToMenus[PrevCount].end());
            }
            CountToMenus[CurrentCount].push_back(Menu);
        }
    }

    for (const int Course : course)
    {
        bool Flag = false;
        for (int Count = CountToMenus.size()-1; Count>=2; Count--)
        {
            for (const string& Menu : CountToMenus[Count])
            {
                if (Menu.size() == Course)
                {
                    answer.push_back(Menu);
                    Flag = true;
                }
            }
            if (Flag) break;
        }
    }

    sort(answer.begin(), answer.end());

    return answer;
}
~~~