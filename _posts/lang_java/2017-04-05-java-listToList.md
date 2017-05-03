---
layout: post
title: java List转List: Lists.transform
category: java
comments: false
---
java支持List到List互转，只要实现transform里面的function函数参数。

    public List<BuInfo> getBusinessUnitByBGId(int bgId) {
        List<Integer> buIds = businessUnitGroupDao.queryBuIdsByBgId(bgId);
        if(CollectionUtils.isEmpty(buIds)) return Collections.emptyList();
        List<BuInfo> buInfos = Lists.transform(buIds, new Function<Integer, BuInfo>() {
            @Override
            public BuInfo apply(Integer buId) {     　//function 左边为原list里面类型  右边为需要转list的类型
                GroupPO groupPO = groupDao.loadGroup(buId);
                BuInfo buInfo = new BuInfo();
                buInfo.setId(buId);
                buInfo.setName(groupPO.getGroupName());
                return buInfo;
            }
        });
        return buInfos;
    }