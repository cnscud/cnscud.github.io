---
layout: post
title: 解析crontab正则,增加+操作
date:   2020-04-20 12:22
description: 
categories: python
comments: true
tags:
- python,crontab
---

以下是使用Python解析crontab时间格式的一个类, 同时minute和hour支持了 + 的操作. 记录一下备忘.

其中的line参数是字符串分拆后的格式, 包含了 "week", "month", "day", "hour", "minute".


```python
#!/usr/bin/env python3
# -*-coding:utf-8-*-

"""
支持crontab表达式的分解, 判断是否符合条件.
@author Felix Zhang
@contact cnscud@gmail.com
@since 2018.1.11
"""

from datetime import datetime


class MatchTimeExp(object):
	TIMEUNIT_FORMAT = {"minute": "%M", "hour": "%H", "day": "%d", "month": "%m", "year": "%Y", "week": "%w"}
	TIMEUNIT_SCALE = {"minute": 60, "hour": 24}

	@staticmethod
	def fire(line, cur_time):
		holders = ["week", "month", "day", "hour", "minute"]
		for h in holders:
			if line[h] is not None and line[h] != "":
				ret = MatchTimeExp.matchtime(line[h], h, cur_time)
				if ret != 0:
					return ret

		return 0

	@staticmethod
	def matchtime(exp, timeunit, thetime):
		"""支持的格式: *  5 */5  */5+2  1,2,3 	5-22 	5-8,11-15
			@:type thetime datetime
			@:return match=0 , not match =1 , error = -1
		"""
		assert isinstance(thetime, datetime)

		exp = exp.replace(" ", "")

		digit_exp = exp.replace(",", "").replace("-", "").replace("/", "").replace("*", "").replace("+", "")
		if digit_exp != "" and not digit_exp.isdigit():
			return -1

		# 分解逗号
		nodes = exp.split(",")
		if len(nodes) > 1:
			for node in nodes:
				if node != "" and MatchTimeExp.__matchtime_one(node, timeunit, thetime) == 0:
					return 0
			return 1
		else:
			return MatchTimeExp.__matchtime_one(exp, timeunit, thetime)

	@staticmethod
	def __check_plusexp(step, timeunit, curtimenode):
		""" 支持+ 的特殊语法"""

		# 仅支持
		if timeunit not in ("minute", "hour"):
			return -1

		parts = step.strip().split("+")
		if len(parts) == 2 and parts[0].strip().isdigit() and parts[1].strip().isdigit():
			mystep = int(parts[0])
			plusvalue = int(parts[1])
			if plusvalue >= mystep:
				return -1

			timenode = curtimenode - plusvalue
			if timenode < 0:
				timenode += MatchTimeExp.TIMEUNIT_SCALE.get(timeunit)

			if timenode % mystep == 0:
				return 0
		else:
			return -1

		return 1

	@staticmethod
	def __matchtime_one(exp, timeunit, thetime):
		if exp == "*":
			return 0

		if exp == "" or exp is None:
			return 1

		curtimenode = int(thetime.strftime(MatchTimeExp.TIMEUNIT_FORMAT.get(timeunit)))

		if exp == str(curtimenode):
			return 0

		patternfind = False

		items = exp.split('/')
		if len(items) == 2 and items[0] == "*":
			patternfind = True
			step = items[1]
			if step.isdigit():
				if curtimenode % int(step) == 0:
					return 0
			else:
				return MatchTimeExp.__check_plusexp(step, timeunit, curtimenode)
		elif len(items) > 1:
			return -1

		# # 逗号
		# nodes = exp.split(",")
		# if len(nodes) > 0:
		# 	for node in nodes:
		# 		if node.strip() == str(curtimenode):
		# 			return 0

		# 减号:表示范围
		nodes = exp.split("-")
		if len(nodes) > 1:
			patternfind = True
			if len(nodes) == 2 and nodes[0].strip().isdigit() and nodes[1].strip().isdigit():
				if int(nodes[0].strip()) <= curtimenode <= int(nodes[1].strip()):
					return 0
			else:
				return -1

		if not patternfind and not exp.isdigit():
			return -1

		return 1


def main():
	thetime = datetime.now()
	thetime = thetime.replace(minute=5)

	# 测试分钟
	test("*", thetime, 0)
	test("*/5", thetime, 0)
	test("*/3", thetime, 1)
	test("5", thetime, 0)
	test("6", thetime, 1)
	test("5,10,15", thetime, 0)
	test("2,4,6", thetime, 1)
	test("2-6", thetime, 0)
	test("12-25", thetime, 1)

	test("2-6,9-12", thetime, 0)
	test("12-15, 20-23", thetime, 1)

	thetime = datetime.now()
	thetime = thetime.replace(minute=15)
	test("*/5", thetime, 0)
	test("*/3", thetime, 0)
	test("*/7", thetime, 1)
	test("6", thetime, 1)
	test("5,10,15", thetime, 0)
	test("2-6", thetime, 1)
	test("12-25", thetime, 0)
	test("2-6,9-12", thetime, 1)
	test("12-15, 20-23", thetime, 0)

	thetime = datetime.now()
	thetime = thetime.replace(minute=5)
	test("*/7+6", thetime, 1)
	test("*/7+5", thetime, 0)
	test("*/7+2", thetime, 1)

	thetime = datetime.now()
	thetime = thetime.replace(minute=1)
	test("*/7+6", thetime, 1)
	test("*/7+5", thetime, 0)
	test("*/7+2", thetime, 1)

	thetime = datetime.now()
	thetime = thetime.replace(minute=12)
	test("*/7+5", thetime, 0)
	test("*/7+1", thetime, 1)
	test("*/7+9", thetime, -1)

	# wrong exp
	test("a-b", thetime, -1)
	test("a,2", thetime, -1)
	test("*/b", thetime, -1)
	test("*/7+a", thetime, -1)
	test("*/a+b", thetime, -1)

	# , + - / *

	thetime = datetime.now()
	thetime = thetime.replace(minute=12)

	test("12,", thetime, 0)
	test("11,", thetime, 1)
	test(",2", thetime, 1)

	test("3-5-8", thetime, -1)
	test("3-", thetime, -1)
	test("-3", thetime, -1)
	test("5+2", thetime, -1)
	test("/2", thetime, -1)
	test("2/", thetime, -1)
	test("*5", thetime, -1)

	thetime = datetime.now()
	thetime = thetime.replace(hour=8)

	test("*/3+2", thetime, 0, "hour")
	test("*/3+1", thetime, 1, "hour")
	test("*/3+5", thetime, -1, "hour")

	thetime = thetime.replace(hour=1)

	test("*/3+2", thetime, 1, "hour")
	test("*/3+1", thetime, 0, "hour")

	thetime = thetime.replace(day=5)

	test("*/4", thetime, 1, "day")
	test("*/5", thetime, 0, "day")
	test("*/3+2", thetime, -1, "day")

	thetime = datetime.now()
	thetime = thetime.replace(minute=6)

	test_all({"week": "*", "month": "*", "day": "*", "hour": "*", "minute": "*/3"}, thetime, 0)
	test_all({"week": "*", "month": "*", "day": "*", "hour": "*", "minute": "*/4"}, thetime, 1)


def test_all(line, curtime, okresult):
	timeprint = curtime.strftime("%Y-%m-%d %H:%M:%S")
	# minute   hour   day   month   week
	lineprint = " ".join([line["minute"], line["hour"], line["day"], line["month"], line["week"]])
	if MatchTimeExp.fire(line, curtime) == okresult:
		print("pass: matchtime %s check: (%s) is %i in %s" % ("all", lineprint, okresult, timeprint))
	else:
		print("not pass: matchtime %s check (%s) is %i in %s" % ("all", lineprint, okresult, timeprint))


def test(exp, curtime, okresult, unit="minute"):
	timeprint = curtime.strftime("%Y-%m-%d %H:%M:%S")
	if MatchTimeExp.matchtime(exp, unit, curtime) == okresult:
		print("pass: matchtime %s check %s is in %s" % (unit, exp, timeprint))
	else:
		print("not pass: matchtime %s check %s is in %s" % (unit, exp, timeprint))


if __name__ == '__main__':
	main()

```