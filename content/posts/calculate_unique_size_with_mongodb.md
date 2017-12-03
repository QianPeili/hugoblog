---
title: "用mongodb统计去重后的数量"
date: 2017-03-16T18:11:29+08:00
category: 
 - 小技巧
tags: 
 - mongodb
---


假设有一个活动记录表，记录了玩家的参与信息，每个玩家有多条记录，格式如下

	{
		_id: ObjectId(),
        uid: 1014,
        score: 10
	}

要求统计参与活动的玩家中，分数大于等于10的人数。

	var num = db.activity_log.aggregate([
		{$match: {"score": {"$ge": 10}}},  # 筛选
		{$group: {"_id": null, "users": {$addToSet: "$uid"}}},    # 将玩家ID放入一个集合，作去重
		{"$project": {"_id": {"$size": "$users"}}}    # 求集合大小
	]);

得出结果`{"_id": 46}`