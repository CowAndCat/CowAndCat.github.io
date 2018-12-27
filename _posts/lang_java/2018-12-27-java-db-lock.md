---
layout: post
title: Java 利用数据库解决分布式并发问题
category: java
comments: false
---

## 一、问题背景

多个集群同时接受到请求，操作同一个数据库的数据，需要保证并发安全性，不产生脏数据。

## 二、解决方法

由于是分布式环境，集群之间能沟通的渠道只能是它们共同资源。借鉴CAS思想，设计一个基于数据库的分布式同步方案。

### 2.1 关键代码

通过过期时间和状态来判断是否能拿到锁（相当于compare），通过更新过期时间来获取锁（swap）。如果后续没有操作，锁会自动过期并释放。

Mapper代码（加锁和释放）：

    @Update('''
        UPDATE
            `$_parameter.tableName`
        SET
            expire_time = @{expireTime},
            update_time = NOW()
        WHERE
            account_id = @{accountId}
            AND cluster_id = @{clusterId}
            AND status = @{status}
            AND (failure_msg IS NULL OR failure_msg = '')
            AND expire_time <= NOW()
    ''')
    int lockClusterStatus(@Param("dataSourceInfo") DataSourceInfo dataSourceInfo,
            @Param("accountId") String accountId, @Param("clusterId") String clusterId,
            @Param("status") ClusterStatus status, @Param("expireTime") Date expireTime,
            @Param("tableName") String tableName);

    @Update('''
        UPDATE
            `$_parameter.tableName`
        SET
            status = @{status},
            failure_msg = @{failureMsg},
            expire_time = @{expireTime},
            update_time = NOW()
        WHERE
            account_id = @{accountId}
            AND cluster_id = @{clusterId}
    ''')
    int updateClusterStatus(@Param("dataSourceInfo") DataSourceInfo dataSourceInfo,
            @Param("accountId") String accountId, @Param("clusterId") String clusterId,
            @Param("status") ClusterStatus status, @Param("failureMsg") String failureMsg,
            @Param("expireTime") Date expireTime, @Param("tableName") String tableName);

Manager的代码：

    // try lock
    private Cluster tryLockCluster(final String accountId, final String clusterId,
            final ClusterStatus status, final int timeoutSeconds) {
        LocalDateTime expireDateTime = LocalDateTime.now().plusSeconds(timeoutSeconds);
        int count = clusterMapper.lockClusterStatus(dataSourceInfo, accountId, clusterId, status,
                expireDateTime.toDate());
        if (count > 0) {
            return clusterMapper.getClusterById(dataSourceInfo, accountId, clusterId)
                    .toClusterInfo(getConfigParameterType(), getClusterParameterType());
        } else {
            log.info(new LogMessageBuilder()
                    .withMessage("Fail to lock cluster.")
                    .withParameter("clusterId", clusterId)
                    .withParameter("status", status)
                    .build());
            return null;
        }
    }

    private void unlockCluster(Cluster clusterInfo) {
        LocalDateTime unlockTime = LocalDateTime.now().minusSeconds(1);
        clusterMapper.updateClusterStatus(dataSourceInfo, clusterInfo.getAccountId(), 
                clusterInfo.getClusterId(), clusterInfo.getStatus(), clusterInfo.getFailureMsg(), 
                unlockTime.toDate());
    }

释放锁的时候，将过期时间设置成当前时间的前一秒钟，下次有谁tryLockCluster就能成功。

使用数据库锁的代码示例：

    private void shrinkCluster(final String accountId, final String clusterId) {
        Cluster clusterInfo = tryLockCluster(
                accountId, clusterId, ClusterStatus.TO_SHRINK, LOCK_TIMEOUT_SECOND);
        if (clusterInfo != null) {
            try {
                componentService.shrinkClusterComponents(clusterInfo);
                clusterInfo.setStatus(ClusterStatus.STARTED);
                clusterInfo.setFailureMsg(StringUtils.EMPTY);
                return;
            } catch (Throwable e) {
                log.error(new LogMessageBuilder()
                        .withMessage("Fail to shrink cluster.")
                        .withParameter("accountId", accountId)
                        .withParameter("clusterId", clusterId)
                        .build(), e);
                clusterInfo.setFailureMsg(e.toString());
                throw new RuntimeException(e);
            } finally {
                unlockCluster(clusterInfo);
            }
        }
    }

### 2.2 方案评价

总得来说，方案比较简单。当拿不到锁的时候，并不会循环获取，也不会有被唤醒的过程，全程都是主动去获取锁，获取失败就结束操作。  
操作的重试由上层的schedule进行调度。

在获取锁的时候，加上了status这个参数，这样能保证cluster集群的状态迁移的拓扑顺序。