---
title: C#字符串Split方法分析
date: 2017-08-14 00:14:15
tags:
    - .net
    - c#
---

参考了C#源代码中String.Split的实现。

``` csharp
using System;
using System.Diagnostics.Contracts;
using System.Linq;

namespace MySplit
{
    class Program
    {
        static void Main(string[] args)
        {
            string[] result = split("we are families.",
                new char[] { 'e', 'a', 'e' },
                2,
                StringSplitOptions.RemoveEmptyEntries);

            result.ToList().ForEach(s => Console.Write("[{0}] ", s));
        }

        static string[] split(
            string str,
            char[] separator,
            int? count = Int32.MaxValue,
            StringSplitOptions options = StringSplitOptions.None)
        {
            return splitInternal(str, separator, count.Value, options);
        }

        static string[] splitInternal(
            string str,
            char[] separator,
            int count,
            StringSplitOptions options)
        {
            if (count < 0)
                throw new ArgumentOutOfRangeException("count");

            bool omitEmptyEntries = (options == StringSplitOptions.RemoveEmptyEntries);

            if (count == 0 || str.Length == 0)
            {
                return new string[0];
            }

            int[] sepList = new int[str.Length];
            int numReplaces = makeSeparatorList(str, separator, ref sepList);

            if (0 == numReplaces || count == 1)
            {
                return new string[] { str };
            }

            if (omitEmptyEntries)
            {
                return internalSplitOmitEmptyEntries(str, sepList, null, numReplaces, count);
            }
            else
            {
                return internalSplitKeepEmptyEntries(str, sepList, null, numReplaces, count);
            }
        }

        static int makeSeparatorList(string str, char[] separator, ref int[] sepList)
        {
            int foundCount = 0;

            if (separator == null || separator.Length == 0)
            {
                for (int i = 0; i < str.Length && foundCount < separator.Length; i++)
                {
                    if (Char.IsWhiteSpace(str[i]))
                    {
                        sepList[foundCount++] = i;
                    }
                }
            }
            else
            {
                int sepListCount = sepList.Length;
                int sepCount = separator.Length;

                for (int i = 0; i < str.Length && foundCount < sepListCount; i++)
                {
                    for (int j = 0; j < sepCount; j++)
                    {
                        if (str[i] == separator[j])
                        {
                            sepList[foundCount++] = i;
                        }
                    }

                }
            }

            return foundCount;
        }

        static string[] internalSplitOmitEmptyEntries(
            string str,
            int[] sepList,
            int[] lengthList,
            int numReplaces,
            int count)
        {
            Contract.Requires(numReplaces >= 0);
            Contract.Requires(count >= 2);
            Contract.Ensures(Contract.Result<string[]>() != null);

            int maxItems = (numReplaces < count) ? (numReplaces + 1) : count;
            string[] splitStrings = new string[maxItems];

            int currentIndex = 0;
            int arrIndex = 0;

            for (int i = 0; i < numReplaces && currentIndex < str.Length; i++)
            {
                if (sepList[i] - currentIndex > 0)
                {
                    splitStrings[arrIndex++] =
                        str.Substring(currentIndex, sepList[i] - currentIndex);
                }

                currentIndex = sepList[i] + ((lengthList == null) ? 1 : lengthList[i]);

                if (arrIndex == count - 1)
                {
                    while (i < numReplaces - 1 && currentIndex == sepList[++i])
                    {
                        currentIndex += ((lengthList == null) ? 1 : lengthList[i]);
                    }
                    break;
                }
            }

            Contract.Assert(arrIndex < maxItems);

            if (currentIndex < str.Length)
            {
                splitStrings[arrIndex++] = str.Substring(currentIndex);
            }

            string[] stringArray = splitStrings;
            if (arrIndex != maxItems)
            {
                stringArray = new string[arrIndex];
                for (int j = 0; j < arrIndex; j++)
                    stringArray[j] = splitStrings[j];
            }

            return stringArray;
        }
        static string[] internalSplitKeepEmptyEntries(
            string str,
            int[] sepList,
            int[] lengthList,
            int numReplacese,
            int count)
        {
            Contract.Requires(numReplacese >= 0);
            Contract.Requires(count >= 2);
            Contract.Ensures(Contract.Result<string[]>() != null);

            return null;
        }
    }
}
```
