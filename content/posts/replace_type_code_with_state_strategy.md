---
title: "Replace Type Code With State/Strategy"
date: 2016-12-26T20:55:47+08:00
categories: 
 - coding
---

今天在阅读《重构：改善既有代码的设计》第一个案例中，遇到了一个**replace type code with state/strategy**的概念，不是很了解，
但大概懂了意思。决定用`python`实现一下其代码。

觉得主要思想是：

一个对象(`Movie`)有一个属性(`price_code`)，这个属性的不同会导致对象的方法(`get_charge`)返回不同的结果。

最开始的想法肯定是用subclass进行分类，但是一部电影可以更换`price_code`，一个对象没法更换其所属类，所以直接建立一个`price`对象作为属性，改变`price_code`时，即改变了`price`对象。


重构前的代码：

	class Movie(object):
	
	    REGULAR = 0
	    NEW_RELEASE = 1
	    CHILDRENS = 2
	
	    def __init__(self, title, price_code):
	        self.title = title
	        self.price_code = price_code
	
	
	    def get_charge(self, days_rented):
	        result = 0
	
	        if self.price_code == self.REGULAR:
	            result += 2
	            if days_rented > 2:
	                result += (days_rented - 2) * 1.5
	        elif self.price_code == self.NEW_RELEASE:
	            result += days_rented * 3
	        elif self.price_code == self.CHILDRENS:
	            result += 1.5
	            if days_rented > 3:
	                result += (days_rented - 3) * 1.5
	
	        return result
	
	
	movie = Movie("Some Movie", 1)
	print movie.get_charge(5)


第一步：

	class Movie(object):
	
	    REGULAR = 0
	    NEW_RELEASE = 1
	    CHILDRENS = 2
	
	    def __init__(self, title, price_code):
	        self.title = title
	        self.price = Price(price_code)
	
	
	    def get_charge(self, days_rented):
	        return self.price.get_charge(days_rented)
	
	
	class Price(object):
	
	    _price_code = 0
	
	    def __init__(self, price_code):
	        self.price_code = price_code
	
	    @property
	    def price_code(self):
	        return self._price_code
	
	    @price_code.setter
	    def price_code(self, value):
	        self._price_code = value
	
	    def get_charge(self, days_rented):
	        result = 0
	
	        if self.price_code == Movie.REGULAR:
	            result += 2
	            if days_rented > 2:
	                result += (days_rented - 2) * 1.5
	        elif self.price_code == Movie.NEW_RELEASE:
	            result += days_rented * 3
	        elif self.price_code == Movie.CHILDRENS:
	            result += 1.5
	            if days_rented > 3:
	                result += (days_rented - 3) * 1.5
	
	        return result
	
	movie = Movie("Some Movie", 1)
	print movie.get_charge(5)


第二步：

	class Movie(object):
	
	    REGULAR = 0
	    NEW_RELEASE = 1
	    CHILDRENS = 2
	
	    _price = None
	
	    def __init__(self, title, price_code):
	        self.title = title
	        self.price = price_code
	
	    @property
	    def price(self):
	        return self._price
	
	    @price.setter
	    def price(self, value):
	        if value == self.REGULAR:
	            self._price = RegularPrice()
	        elif value == self.NEW_RELEASE:
	            self._price = NewReleasePrice()
	        elif value == self.CHILDRENS:
	            self._price = ChildrenPrice()
	
	
	    def get_charge(self, days_rented):
	        return self._price.get_charge(days_rented)
	
	
	class Price(object):
	
	    @property
	    def price_code(self):
	        return self._price_code
	
	    @price_code.setter
	    def price_code(self, value):
	        self._price_code = value
	
	    def get_charge(self, days_rented):
	        pass
	
	
	class RegularPrice(Price):
	
	    def get_charge(self, days_rented):
	        result = 2
	        if days_rented > 2:
	            result += (days_rented - 2) * 1.5
	
	
	class NewReleasePrice(Price):
	
	    def get_charge(self, days_rented):
	        return days_rented * 3
	
	
	class ChildrenPrice(Price):
	
	    def get_charge(self, days_rented):
	        result = 1.5
	        if days_rented > 3:
	            result += (days_rented - 3) * 1.5
	        return result
	
	
	movie = Movie("Some Movie", 1)
	print movie.get_charge(5)
