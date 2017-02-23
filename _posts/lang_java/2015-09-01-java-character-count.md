---
layout: post
title: Java统计字符串中汉字,英文,数字,特殊符号个数
category: java
comments: false
---
## Java中的字符统计
通过正则表达式：

	public void characterCount(){
		String str = "a12中国3@b&4语*言3c";

        String E1 = "[\u4e00-\u9fa5]";// 中文
        String E2 = "[a-zA-Z]";// 英文
        String E3 = "[0-9]";// 数字

        int chineseCount = 0;
        int englishCount = 0;
        int numberCount = 0;

        String temp;
        for (int i = 0; i < str.length(); i++)
        {
            temp = String.valueOf(str.charAt(i));
            if (temp.matches(E1))
            {
                chineseCount++;
            }
            if (temp.matches(E2))
            {
                englishCount++;
            }
            if (temp.matches(E3))
            {
                numberCount++;
            }
        }
        System.out.println("汉字数:" + chineseCount);
        System.out.println("英文数:" + englishCount);
        System.out.println("数字数:" + numberCount);
        System.out.println("特殊字符:" + (str.length() - (chineseCount + englishCount + numberCount)));
	}

记住[\u4e00-\u9fa5], String.matches(regex)

或者这样统计汉字：

	public static int countChinese(String text) {
		int result = 0;
		char ch;
		for (int i = 0; i < text.length(); i++) {
			ch = text.charAt(i);
			// 判断Unicode
			if (ch >= 19968 && ch <= 64041) {
				result++; // 汉字
			}
		}
		return result;
	}
