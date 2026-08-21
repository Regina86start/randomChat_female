# 50-女坐等摸底 SQL

> **日期先改**：当日 `20260817`；近一月 `20260718`～`20260817`（先按天算，再对天数取 AVG）。  
> **近一月控扫描**：埋点一律 `server_log_data`（按 `ds` 分区 + `log_id='333'`）。**当日和近一月分档都用** `dws_user_f_d.history_earn`（行为当日）。  
> **§7 / §8 控扫描**：333 先按 `type=1` 或 `type=2且endCode IN (1,2,3)` 收窄并配对，**配对后再 JOIN 活跃女分档**。禁止 `server_log_data` 直接 JOIN 整月 `active_w`。  
> **§7.2/7.3、§8.2/8.3、§11.2/11.3 拆段**：整月 333 配对或多表交叉会 `ODPS-0020071`。上半月 `20260718`～`20260801`（15 天），下半月 `20260802`～`20260817`（16 天）。整月日均 = `(上半月日均×上半月统计天数 + 下半月日均×下半月统计天数) / 两段统计天数之和`。不要把两段日均直接相加。

## 口径

| 项 | 定义 |
|---|---|
| 分母 | 当天活跃女：`t_userlog_retained` ∩ `dws_user_f_d.sex=0` |
| 分档 | 当日 / 近一月均用 `dws_user_f_d.history_earn`（行为当日）。档位：50-女 &lt;5万；50-100女 5万～&lt;10万；100+女 ≥10万；`history_earn` 为空才算财富缺失 |
| 坐等 | `chamet_data.server_log_data`（按 `ds` 分区），`log_id='333'` 且 `extra.type=1`。结束码主路径 1/2/3，不计入坐等人数 |
| 被打到 | `dwd_video_detail_i_d`：被叫=该坐等女，主叫男，`receive_type=12`（速配坐等）。不用 1（坐等在线，老口径）。**不筛 call_type** |
| 接通 | 被打到且 `call_time>0` 或 `valid_time>0` |
| 开播 | 单人直播 `t_room.type=2` + 当日 `t_room_statistic`，有记录即算 |
| 坐等接通赚取 | 仅上述接通订单的 `integral+integral_give+gift_integral`，不含当天其他视频聊 |
| 一轮坐等 | 同一 `extra.bizCode` 上 `type=1` 开始 + `type=2` 结束（同 `userid`、同 `ds`）。跨天对不上，计入开始、不计入配对 |
| 坐等时长 | `extra.createTime` 间隔秒：`DATEDIFF(结束, 开始, 'ss')`。只留 `1～7200`（≤0 或 &gt;2 小时剔除）。均值 + 中位数都出；中位数用 `PERCENTILE_APPROX`。`endCode=1` 的时长是等到被匹配走，不是通话时长 |
| 结束码 | 只认 1 成功接通结束 / 2 主动挂断 / 3 心跳超时 |
| 其他视频聊 | 坐等女当天被男打到且 `receive_type≠12`（约聊/主页/直播态等一律归「其他」） |
| 注册后天数 | 只拆 50-：D0 注册当天；D1-7；D8+。`create_time` 解析不出单独「注册日缺失」 |
| 漏斗包含 | 接通 ⊂ 被打到 ⊂ 坐等 ⊂ 活跃女 |

本套**不能**证明首页随机匹配弹窗导致坐等，只能看坐等结果。

产出列名全中文。埋点一律 `server_log_data`，必带 `ds`+`log_id`。禁止 `CROSS JOIN`；日均缺档用假键 `k=1` + `MAPJOIN` 补 0。所有率/占比为小数（如 0.1467 = 14.67%），未乘 100。

---

## 0. 前置：333 extra（已跑过可跳）

`MatchChatEndCodeEnum`（`type=2` 结束坐等才有 `endCode`）。女坐等主路径目前只有 **1 / 2 / 3**：

| 结束码 | 枚举 | 含义 |
|---|---|---|
| 1 | SUCCESS_MATCH | 速配成功，接通后结束女方坐等 |
| 2 | HANG_UP | 主动挂断（客户端 `endQuickPairForFemale`，未传则默认 2） |
| 3 | HEART_BEAT_OVER | 坐等心跳超时，服务端强制下榜 |
| 4～7 | 进队列失败 / 客户端心跳 / 客户端异常 / 男离线 | 枚举有，女坐等 333 主路径基本不用 |

```sql
--确认 type=1 开始（结束码空）、type=2 结束；女坐等主路径结束码应只有 1/2/3
SELECT  GET_JSON_OBJECT(extra, '$.type')     AS `坐等类型`
        ,GET_JSON_OBJECT(extra, '$.endCode') AS `结束码`
        ,COUNT(1)                             AS `条数`
        ,COUNT(DISTINCT userid)               AS `人数`
FROM    chamet_data.server_log_data
WHERE   ds = '20260817'
AND     log_id = '333'
AND     (
            CAST(GET_JSON_OBJECT(extra, '$.type') AS BIGINT) = 1
            OR CAST(GET_JSON_OBJECT(extra, '$.endCode') AS BIGINT) IN (1, 2, 3)
        )
GROUP BY GET_JSON_OBJECT(extra, '$.type')
         ,GET_JSON_OBJECT(extra, '$.endCode')
ORDER BY `条数` DESC
;
```

后续坐等人数一律用 **`type=1` 去重**，不要把 type=2 加进去。看结束原因时只用 **endCode IN (1,2,3)**。

---

## 1. 看坐等分布

问：去坐等的女是哪一档。看坐等率、坐等女结构是不是堆在 50-。

### 1.1 当日（20260817）

```sql
--20260817当日：活跃女坐等分布（50-/50-100/100+，分档用 dws.history_earn）
WITH active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  DISTINCT CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    JOIN    active_w a
    ON      CAST(s.userid AS BIGINT) = a.userid
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
)
SELECT  a.赚取分档 AS `赚取分档`
        ,COUNT(1) AS `活跃女人数`
        ,COUNT(w.userid) AS `当天坐等人数`
        ,ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4) AS `当天坐等率`
        ,ROUND(COUNT(w.userid) * 1.0 / SUM(COUNT(w.userid)) OVER (), 4) AS `占全部坐等女`
FROM    active_w a
LEFT JOIN wait_u w
ON      a.userid = w.userid
GROUP BY a.赚取分档
ORDER BY CASE a.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

### 1.2 近一月日均（20260718～20260817）

```sql
--近一月日均(20260718~20260817)：活跃女坐等分布（埋点用 server_log_data，分档用 dws.history_earn）
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, daily AS (
    SELECT  a.ds
            ,a.赚取分档
            ,COUNT(1) AS 活跃女人数
            ,COUNT(w.userid) AS 坐等人数
            ,CASE WHEN COUNT(1) = 0 THEN 0
                  ELSE ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4)
             END AS 坐等率
    FROM    active_w a
    LEFT JOIN wait_u w
    ON      a.ds = w.ds
    AND     a.userid = w.userid
    GROUP BY a.ds, a.赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    daily
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档
    UNION ALL SELECT 1, '50-100女'
    UNION ALL SELECT 1, '100+女'
    UNION ALL SELECT 1, '财富缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,AVG(NVL(d.活跃女人数, 0)) AS `日均活跃女人数`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.坐等率, 0)) AS `日均坐等率`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.赚取分档 = d.赚取分档
GROUP BY g.赚取分档
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

---

## 2. 看坐等接通

问：坐等女里，有多少被男打到、有多少接通（有时长）。被打到只计 `receive_type=12`（速配坐等）。

### 2.1 当日（20260817）

```sql
--20260817当日：坐等被打到 / 接通（不限 call_type）
WITH active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  DISTINCT CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    JOIN    active_w a
    ON      CAST(s.userid AS BIGINT) = a.userid
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
)
, hit_u AS (
    SELECT  DISTINCT v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
)
, conn_u AS (
    SELECT  DISTINCT v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
)
SELECT  a.赚取分档 AS `赚取分档`
        ,COUNT(w.userid) AS `当天坐等人数`
        ,COUNT(h.userid) AS `坐等被男打到人数`
        ,COUNT(c.userid) AS `坐等接通人数`
        ,CASE WHEN COUNT(w.userid) = 0 THEN 0
              ELSE ROUND(COUNT(h.userid) * 1.0 / COUNT(w.userid), 4)
         END AS `坐等被打到率`
        ,CASE WHEN COUNT(w.userid) = 0 THEN 0
              ELSE ROUND(COUNT(c.userid) * 1.0 / COUNT(w.userid), 4)
         END AS `坐等接通率`
FROM    active_w a
LEFT JOIN wait_u w ON a.userid = w.userid
LEFT JOIN hit_u h ON a.userid = h.userid
LEFT JOIN conn_u c ON a.userid = c.userid
GROUP BY a.赚取分档
ORDER BY CASE a.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

### 2.2 近一月日均（20260718～20260817）

```sql
--近一月日均(20260718~20260817)：坐等被打到 / 接通
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, hit_u AS (
    SELECT  v.ds
            ,v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    GROUP BY v.ds, v.called_id
)
, conn_u AS (
    SELECT  v.ds
            ,v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
    GROUP BY v.ds, v.called_id
)
, daily AS (
    SELECT  a.ds
            ,a.赚取分档
            ,COUNT(w.userid) AS 坐等人数
            ,COUNT(h.userid) AS 被打到人数
            ,COUNT(c.userid) AS 接通人数
            ,CASE WHEN COUNT(w.userid) = 0 THEN 0
                  ELSE ROUND(COUNT(h.userid) * 1.0 / COUNT(w.userid), 4)
             END AS 坐等被打到率
            ,CASE WHEN COUNT(w.userid) = 0 THEN 0
                  ELSE ROUND(COUNT(c.userid) * 1.0 / COUNT(w.userid), 4)
             END AS 坐等接通率
    FROM    active_w a
    LEFT JOIN wait_u w ON a.ds = w.ds AND a.userid = w.userid
    LEFT JOIN hit_u h ON a.ds = h.ds AND a.userid = h.userid
    LEFT JOIN conn_u c ON a.ds = c.ds AND a.userid = c.userid
    GROUP BY a.ds, a.赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    daily
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档
    UNION ALL SELECT 1, '50-100女'
    UNION ALL SELECT 1, '100+女'
    UNION ALL SELECT 1, '财富缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.被打到人数, 0)) AS `日均坐等被男打到人数`
        ,AVG(NVL(d.接通人数, 0)) AS `日均坐等接通人数`
        ,AVG(NVL(d.坐等被打到率, 0)) AS `日均坐等被打到率`
        ,AVG(NVL(d.坐等接通率, 0)) AS `日均坐等接通率`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.赚取分档 = d.赚取分档
GROUP BY g.赚取分档
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

---

## 3. 看坐等接通漏斗

问：从活跃走到接通，掉在哪一层。三档各一条。

```
活跃女 → 当天坐等过 → 坐等中被男打到 → 接通（有时长）
```

### 3.1 当日（20260817）

```sql
--20260817当日：坐等接通漏斗（人数 + 相对上一层 + 占活跃女）
WITH active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  DISTINCT CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    JOIN    active_w a
    ON      CAST(s.userid AS BIGINT) = a.userid
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
)
, hit_u AS (
    SELECT  DISTINCT v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
)
, conn_u AS (
    SELECT  DISTINCT v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
)
SELECT  a.赚取分档 AS `赚取分档`
        ,COUNT(1) AS `活跃女人数`
        ,COUNT(w.userid) AS `坐等人数`
        ,COUNT(h.userid) AS `坐等被男打到人数`
        ,COUNT(c.userid) AS `坐等接通人数`
        ,ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4) AS `活跃到坐等`
        ,CASE WHEN COUNT(w.userid) = 0 THEN 0
              ELSE ROUND(COUNT(h.userid) * 1.0 / COUNT(w.userid), 4)
         END AS `坐等到被打到`
        ,CASE WHEN COUNT(h.userid) = 0 THEN 0
              ELSE ROUND(COUNT(c.userid) * 1.0 / COUNT(h.userid), 4)
         END AS `被打到到接通`
        ,ROUND(COUNT(c.userid) * 1.0 / COUNT(1), 4) AS `活跃到接通`
FROM    active_w a
LEFT JOIN wait_u w ON a.userid = w.userid
LEFT JOIN hit_u h ON a.userid = h.userid
LEFT JOIN conn_u c ON a.userid = c.userid
GROUP BY a.赚取分档
ORDER BY CASE a.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

### 3.2 近一月日均（20260718～20260817）

```sql
--近一月日均(20260718~20260817)：坐等接通漏斗
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, hit_u AS (
    SELECT  v.ds
            ,v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    GROUP BY v.ds, v.called_id
)
, conn_u AS (
    SELECT  v.ds
            ,v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
    GROUP BY v.ds, v.called_id
)
, daily AS (
    SELECT  a.ds
            ,a.赚取分档
            ,COUNT(1) AS 活跃女人数
            ,COUNT(w.userid) AS 坐等人数
            ,COUNT(h.userid) AS 被打到人数
            ,COUNT(c.userid) AS 接通人数
            ,ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4) AS 活跃到坐等
            ,CASE WHEN COUNT(w.userid) = 0 THEN 0
                  ELSE ROUND(COUNT(h.userid) * 1.0 / COUNT(w.userid), 4)
             END AS 坐等到被打到
            ,CASE WHEN COUNT(h.userid) = 0 THEN 0
                  ELSE ROUND(COUNT(c.userid) * 1.0 / COUNT(h.userid), 4)
             END AS 被打到到接通
            ,ROUND(COUNT(c.userid) * 1.0 / COUNT(1), 4) AS 活跃到接通
    FROM    active_w a
    LEFT JOIN wait_u w ON a.ds = w.ds AND a.userid = w.userid
    LEFT JOIN hit_u h ON a.ds = h.ds AND a.userid = h.userid
    LEFT JOIN conn_u c ON a.ds = c.ds AND a.userid = c.userid
    GROUP BY a.ds, a.赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    daily
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档
    UNION ALL SELECT 1, '50-100女'
    UNION ALL SELECT 1, '100+女'
    UNION ALL SELECT 1, '财富缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,AVG(NVL(d.活跃女人数, 0)) AS `日均活跃女人数`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.被打到人数, 0)) AS `日均坐等被男打到人数`
        ,AVG(NVL(d.接通人数, 0)) AS `日均坐等接通人数`
        ,AVG(NVL(d.活跃到坐等, 0)) AS `日均活跃到坐等`
        ,AVG(NVL(d.坐等到被打到, 0)) AS `日均坐等到被打到`
        ,AVG(NVL(d.被打到到接通, 0)) AS `日均被打到到接通`
        ,AVG(NVL(d.活跃到接通, 0)) AS `日均活跃到接通`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.赚取分档 = d.赚取分档
GROUP BY g.赚取分档
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

---

## 4. 看坐等和开播

问：坐等的人当天开没开播。「只坐等不开播」是藏匹配后理论上可能转向开播的池子，不能证明她们会去开播。

### 4.1 当日（20260817）

```sql
--20260817当日：坐等 × 开播四格
WITH active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  DISTINCT CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    JOIN    active_w a
    ON      CAST(s.userid AS BIGINT) = a.userid
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
)
, live_u AS (
    SELECT  DISTINCT b.owner_id AS userid
    FROM    chamet_data.t_room_statistic a
    JOIN    chamet_data.t_room b
    ON      a.room_id = b.room_id
    AND     b.type = 2
    JOIN    active_w w
    ON      b.owner_id = w.userid
    WHERE   a.ds = '20260817'
)
SELECT  a.赚取分档 AS `赚取分档`
        ,COUNT(1) AS `活跃女人数`
        ,COUNT(w.userid) AS `当天坐等人数`
        ,COUNT(l.userid) AS `当天开播人数`
        ,ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4) AS `当天坐等率`
        ,ROUND(COUNT(l.userid) * 1.0 / COUNT(1), 4) AS `当天开播率`
        ,SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NULL THEN 1 ELSE 0 END) AS `只坐等不开播人数`
        ,SUM(CASE WHEN w.userid IS NULL AND l.userid IS NOT NULL THEN 1 ELSE 0 END) AS `只开播不坐等人数`
        ,SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NOT NULL THEN 1 ELSE 0 END) AS `坐等且开播人数`
        ,SUM(CASE WHEN w.userid IS NULL AND l.userid IS NULL THEN 1 ELSE 0 END) AS `坐等开播都无人数`
        ,ROUND(SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(1), 4) AS `只坐等不开播占比`
FROM    active_w a
LEFT JOIN wait_u w ON a.userid = w.userid
LEFT JOIN live_u l ON a.userid = l.userid
GROUP BY a.赚取分档
ORDER BY CASE a.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

### 4.2 近一月日均（20260718～20260817）

```sql
--近一月日均(20260718~20260817)：坐等 × 开播四格
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, live_u AS (
    SELECT  a.ds
            ,b.owner_id AS userid
    FROM    chamet_data.t_room_statistic a
    JOIN    chamet_data.t_room b
    ON      a.room_id = b.room_id
    AND     b.type = 2
    WHERE   a.ds BETWEEN '20260718' AND '20260817'
    GROUP BY a.ds, b.owner_id
)
, daily AS (
    SELECT  a.ds
            ,a.赚取分档
            ,COUNT(1) AS 活跃女人数
            ,COUNT(w.userid) AS 坐等人数
            ,COUNT(l.userid) AS 开播人数
            ,ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4) AS 坐等率
            ,ROUND(COUNT(l.userid) * 1.0 / COUNT(1), 4) AS 开播率
            ,SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NULL THEN 1 ELSE 0 END) AS 只坐等不开播人数
            ,SUM(CASE WHEN w.userid IS NULL AND l.userid IS NOT NULL THEN 1 ELSE 0 END) AS 只开播不坐等人数
            ,SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NOT NULL THEN 1 ELSE 0 END) AS 坐等且开播人数
            ,SUM(CASE WHEN w.userid IS NULL AND l.userid IS NULL THEN 1 ELSE 0 END) AS 坐等开播都无人数
            ,ROUND(SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(1), 4) AS 只坐等不开播占比
    FROM    active_w a
    LEFT JOIN wait_u w ON a.ds = w.ds AND a.userid = w.userid
    LEFT JOIN live_u l ON a.ds = l.ds AND a.userid = l.userid
    GROUP BY a.ds, a.赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    daily
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档
    UNION ALL SELECT 1, '50-100女'
    UNION ALL SELECT 1, '100+女'
    UNION ALL SELECT 1, '财富缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,AVG(NVL(d.活跃女人数, 0)) AS `日均活跃女人数`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.开播人数, 0)) AS `日均开播人数`
        ,AVG(NVL(d.坐等率, 0)) AS `日均坐等率`
        ,AVG(NVL(d.开播率, 0)) AS `日均开播率`
        ,AVG(NVL(d.只坐等不开播人数, 0)) AS `日均只坐等不开播人数`
        ,AVG(NVL(d.只开播不坐等人数, 0)) AS `日均只开播不坐等人数`
        ,AVG(NVL(d.坐等且开播人数, 0)) AS `日均坐等且开播人数`
        ,AVG(NVL(d.坐等开播都无人数, 0)) AS `日均坐等开播都无人数`
        ,AVG(NVL(d.只坐等不开播占比, 0)) AS `日均只坐等不开播占比`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.赚取分档 = d.赚取分档
GROUP BY g.赚取分档
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

---

## 5. 看注册当天（只拆 50-女）

问：50- 里，注册当天是不是更偏坐等、更不爱开播。对上原来「注册新女接聊高、开播弱」。

### 5.1 当日（20260817）

```sql
--20260817当日：50-女 × 是否注册当天（坐等/接通/开播）
WITH active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8) = '20260817'
                THEN '注册当天'
                ELSE '非注册当天'
             END AS 是否注册当天
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
    AND     NVL(u.history_earn, 0) < 50000
)
, wait_u AS (
    SELECT  DISTINCT CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    JOIN    active_w a
    ON      CAST(s.userid AS BIGINT) = a.userid
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
)
, conn_u AS (
    SELECT  DISTINCT v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
)
, live_u AS (
    SELECT  DISTINCT b.owner_id AS userid
    FROM    chamet_data.t_room_statistic a
    JOIN    chamet_data.t_room b
    ON      a.room_id = b.room_id
    AND     b.type = 2
    JOIN    active_w w
    ON      b.owner_id = w.userid
    WHERE   a.ds = '20260817'
)
SELECT  a.是否注册当天 AS `是否注册当天`
        ,COUNT(1) AS `活跃女人数`
        ,COUNT(w.userid) AS `当天坐等人数`
        ,COUNT(c.userid) AS `坐等接通人数`
        ,COUNT(l.userid) AS `当天开播人数`
        ,ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4) AS `当天坐等率`
        ,CASE WHEN COUNT(w.userid) = 0 THEN 0
              ELSE ROUND(COUNT(c.userid) * 1.0 / COUNT(w.userid), 4)
         END AS `坐等接通率`
        ,ROUND(COUNT(l.userid) * 1.0 / COUNT(1), 4) AS `当天开播率`
        ,ROUND(SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(1), 4) AS `只坐等不开播占比`
FROM    active_w a
LEFT JOIN wait_u w ON a.userid = w.userid
LEFT JOIN conn_u c ON a.userid = c.userid
LEFT JOIN live_u l ON a.userid = l.userid
GROUP BY a.是否注册当天
ORDER BY CASE a.是否注册当天 WHEN '注册当天' THEN 1 ELSE 2 END
;
```

### 5.2 近一月日均（20260718～20260817）

```sql
--近一月日均(20260718~20260817)：50-女 × 是否注册当天
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8) = a.ds
                THEN '注册当天'
                ELSE '非注册当天'
             END AS 是否注册当天
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
    AND     NVL(u.history_earn, 0) < 50000
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, conn_u AS (
    SELECT  v.ds
            ,v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
    GROUP BY v.ds, v.called_id
)
, live_u AS (
    SELECT  a.ds
            ,b.owner_id AS userid
    FROM    chamet_data.t_room_statistic a
    JOIN    chamet_data.t_room b
    ON      a.room_id = b.room_id
    AND     b.type = 2
    WHERE   a.ds BETWEEN '20260718' AND '20260817'
    GROUP BY a.ds, b.owner_id
)
, daily AS (
    SELECT  a.ds
            ,a.是否注册当天
            ,COUNT(1) AS 活跃女人数
            ,COUNT(w.userid) AS 坐等人数
            ,COUNT(c.userid) AS 接通人数
            ,COUNT(l.userid) AS 开播人数
            ,ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4) AS 坐等率
            ,CASE WHEN COUNT(w.userid) = 0 THEN 0
                  ELSE ROUND(COUNT(c.userid) * 1.0 / COUNT(w.userid), 4)
             END AS 坐等接通率
            ,ROUND(COUNT(l.userid) * 1.0 / COUNT(1), 4) AS 开播率
            ,ROUND(SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(1), 4) AS 只坐等不开播占比
    FROM    active_w a
    LEFT JOIN wait_u w ON a.ds = w.ds AND a.userid = w.userid
    LEFT JOIN conn_u c ON a.ds = c.ds AND a.userid = c.userid
    LEFT JOIN live_u l ON a.ds = l.ds AND a.userid = l.userid
    GROUP BY a.ds, a.是否注册当天
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    daily
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '注册当天' AS 是否注册当天
    UNION ALL SELECT 1, '非注册当天'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.是否注册当天
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.是否注册当天 AS `是否注册当天`
        ,AVG(NVL(d.活跃女人数, 0)) AS `日均活跃女人数`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.接通人数, 0)) AS `日均坐等接通人数`
        ,AVG(NVL(d.开播人数, 0)) AS `日均开播人数`
        ,AVG(NVL(d.坐等率, 0)) AS `日均坐等率`
        ,AVG(NVL(d.坐等接通率, 0)) AS `日均坐等接通率`
        ,AVG(NVL(d.开播率, 0)) AS `日均开播率`
        ,AVG(NVL(d.只坐等不开播占比, 0)) AS `日均只坐等不开播占比`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.是否注册当天 = d.是否注册当天
GROUP BY g.是否注册当天
ORDER BY CASE g.是否注册当天 WHEN '注册当天' THEN 1 ELSE 2 END
;
```

---

## 6. 看坐等接通的赚取

问：坐等接通这批女，当天靠这些接通单赚了多少。藏匹配的代价。金豆只计坐等接通订单，不含当天其他视频聊。

### 6.1 当日（20260817）

```sql
--20260817当日：坐等接通女的接通单金豆
WITH active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  DISTINCT CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    JOIN    active_w a
    ON      CAST(s.userid AS BIGINT) = a.userid
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
)
, conn_earn AS (
    SELECT  v.called_id AS userid
            ,SUM(NVL(v.integral, 0) + NVL(v.integral_give, 0) + NVL(v.gift_integral, 0)) AS 接通金豆
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
    GROUP BY v.called_id
)
SELECT  a.赚取分档 AS `赚取分档`
        ,COUNT(e.userid) AS `坐等接通人数`
        ,SUM(CASE WHEN e.接通金豆 > 0 THEN 1 ELSE 0 END) AS `接通有赚取人数`
        ,SUM(NVL(e.接通金豆, 0)) AS `接通金豆合计`
        ,CASE WHEN COUNT(e.userid) = 0 THEN 0
              ELSE ROUND(SUM(NVL(e.接通金豆, 0)) * 1.0 / COUNT(e.userid), 2)
         END AS `接通人均金豆`
FROM    active_w a
LEFT JOIN conn_earn e
ON      a.userid = e.userid
GROUP BY a.赚取分档
ORDER BY CASE a.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

> `COUNT(e.userid)` 在 LEFT JOIN 后只统计有接通的人；无接通的档人均为 0。

### 6.2 近一月日均（20260718～20260817）

```sql
--近一月日均(20260718~20260817)：坐等接通女的接通单金豆
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, conn_earn AS (
    SELECT  v.ds
            ,v.called_id AS userid
            ,SUM(NVL(v.integral, 0) + NVL(v.integral_give, 0) + NVL(v.gift_integral, 0)) AS 接通金豆
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
    GROUP BY v.ds, v.called_id
)
, daily AS (
    SELECT  a.ds
            ,a.赚取分档
            ,COUNT(e.userid) AS 接通人数
            ,SUM(CASE WHEN e.接通金豆 > 0 THEN 1 ELSE 0 END) AS 接通有赚取人数
            ,SUM(NVL(e.接通金豆, 0)) AS 接通金豆合计
            ,CASE WHEN COUNT(e.userid) = 0 THEN 0
                  ELSE ROUND(SUM(NVL(e.接通金豆, 0)) * 1.0 / COUNT(e.userid), 2)
             END AS 接通人均金豆
    FROM    active_w a
    LEFT JOIN conn_earn e
    ON      a.ds = e.ds
    AND     a.userid = e.userid
    GROUP BY a.ds, a.赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    daily
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档
    UNION ALL SELECT 1, '50-100女'
    UNION ALL SELECT 1, '100+女'
    UNION ALL SELECT 1, '财富缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,AVG(NVL(d.接通人数, 0)) AS `日均坐等接通人数`
        ,AVG(NVL(d.接通有赚取人数, 0)) AS `日均接通有赚取人数`
        ,AVG(NVL(d.接通金豆合计, 0)) AS `日均接通金豆合计`
        ,AVG(NVL(d.接通人均金豆, 0)) AS `日均接通人均金豆`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.赚取分档 = d.赚取分档
GROUP BY g.赚取分档
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

---

## 7. 看坐等结束原因和时长

问：每一轮坐等怎么结束（被接走 / 自己挂 / 心跳超时被踢），等多久。人数仍用 type=1 去重，本段看轮次。等待均值、中位数都出。

### 7.1 当日（20260817）

```sql
--20260817当日：分档 × 结束码 + 坐等时长（333 先配对再 JOIN 分档）
WITH ev AS (
    SELECT  CAST(s.userid AS BIGINT) AS userid
            ,GET_JSON_OBJECT(s.extra, '$.bizCode') AS biz_code
            ,CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) AS wait_type
            ,CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) AS end_code
            ,TO_DATE(GET_JSON_OBJECT(s.extra, '$.createTime'), 'yyyy-mm-dd hh:mi:ss') AS evt_time
    FROM    chamet_data.server_log_data s
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     GET_JSON_OBJECT(s.extra, '$.bizCode') IS NOT NULL
    AND     (
                CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
                OR (
                    CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 2
                    AND CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) IN (1, 2, 3)
                )
            )
)
, start_u AS (
    SELECT  userid
            ,biz_code
            ,MIN(evt_time) AS start_time
    FROM    ev
    WHERE   wait_type = 1
    GROUP BY userid
             ,biz_code
)
, end_u AS (
    SELECT  userid
            ,biz_code
            ,end_time
            ,end_code
    FROM    (
                SELECT  userid
                        ,biz_code
                        ,evt_time AS end_time
                        ,end_code
                        ,ROW_NUMBER() OVER (PARTITION BY userid, biz_code ORDER BY evt_time DESC) AS rn
                FROM    ev
                WHERE   wait_type = 2
            ) t
    WHERE   rn = 1
)
, sess AS (
    SELECT  s.userid
            ,e.end_code
            ,DATEDIFF(e.end_time, s.start_time, 'ss') AS wait_sec
    FROM    start_u s
    JOIN    end_u e
    ON      s.userid = e.userid
    AND     s.biz_code = e.biz_code
    WHERE   DATEDIFF(e.end_time, s.start_time, 'ss') BETWEEN 1 AND 7200
)
, active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
)
, start_by AS (
    SELECT  a.赚取分档
            ,COUNT(1) AS 开始轮次
            ,COUNT(DISTINCT s.userid) AS 开始人数
    FROM    start_u s
    JOIN    active_w a
    ON      s.userid = a.userid
    GROUP BY a.赚取分档
)
, code_by AS (
    SELECT  a.赚取分档
            ,s.end_code
            ,COUNT(1) AS 配对轮次
            ,COUNT(DISTINCT s.userid) AS 人数
            ,ROUND(AVG(s.wait_sec), 2) AS 等待均值秒
            ,ROUND(PERCENTILE_APPROX(CAST(s.wait_sec AS DOUBLE), 0.5), 2) AS 等待中位数秒
    FROM    sess s
    JOIN    active_w a
    ON      s.userid = a.userid
    GROUP BY a.赚取分档
             ,s.end_code
)
SELECT  c.赚取分档 AS `赚取分档`
        ,c.end_code AS `结束码`
        ,CASE c.end_code
             WHEN 1 THEN '1-成功接通结束'
             WHEN 2 THEN '2-主动挂断'
             WHEN 3 THEN '3-心跳超时'
         END AS `结束原因`
        ,NVL(b.开始轮次, 0) AS `坐等轮次`
        ,NVL(b.开始人数, 0) AS `坐等人数`
        ,c.配对轮次 AS `该结束码轮次`
        ,c.人数 AS `该结束码人数`
        ,SUM(c.配对轮次) OVER (PARTITION BY c.赚取分档) AS `配对轮次合计`
        ,CASE WHEN NVL(b.开始轮次, 0) = 0 THEN 0
              ELSE ROUND(SUM(c.配对轮次) OVER (PARTITION BY c.赚取分档) * 1.0 / b.开始轮次, 4)
         END AS `配对成功率`
        ,ROUND(c.配对轮次 * 1.0 / SUM(c.配对轮次) OVER (PARTITION BY c.赚取分档), 4) AS `占配对轮次`
        ,c.等待均值秒 AS `等待均值秒`
        ,c.等待中位数秒 AS `等待中位数秒`
FROM    code_by c
LEFT JOIN start_by b
ON      c.赚取分档 = b.赚取分档
ORDER BY CASE c.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
         ,c.end_code
;
```

> 配对只在同一 `ds` 内；跨天开始/结束对不上，会压低配对成功率。时长 ≤0 或 &gt;7200 秒不计入配对。`endCode=1` 的等待不是通话时长。

### 7.2 上半月日均（20260718～20260801）

整月拆两段，避免 `ODPS-0020071`。本段 15 天。

```sql
--上半月日均(20260718~20260801)：分档 × 结束码 + 坐等时长（333 先配对再 JOIN 分档）
WITH ev AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
            ,GET_JSON_OBJECT(s.extra, '$.bizCode') AS biz_code
            ,CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) AS wait_type
            ,CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) AS end_code
            ,TO_DATE(GET_JSON_OBJECT(s.extra, '$.createTime'), 'yyyy-mm-dd hh:mi:ss') AS evt_time
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260801'
    AND     s.log_id = '333'
    AND     GET_JSON_OBJECT(s.extra, '$.bizCode') IS NOT NULL
    AND     (
                CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
                OR (
                    CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 2
                    AND CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) IN (1, 2, 3)
                )
            )
)
, start_u AS (
    SELECT  ds
            ,userid
            ,biz_code
            ,MIN(evt_time) AS start_time
    FROM    ev
    WHERE   wait_type = 1
    GROUP BY ds
             ,userid
             ,biz_code
)
, end_u AS (
    SELECT  ds
            ,userid
            ,biz_code
            ,end_time
            ,end_code
    FROM    (
                SELECT  ds
                        ,userid
                        ,biz_code
                        ,evt_time AS end_time
                        ,end_code
                        ,ROW_NUMBER() OVER (PARTITION BY ds, userid, biz_code ORDER BY evt_time DESC) AS rn
                FROM    ev
                WHERE   wait_type = 2
            ) t
    WHERE   rn = 1
)
, sess AS (
    SELECT  s.ds
            ,s.userid
            ,e.end_code
            ,DATEDIFF(e.end_time, s.start_time, 'ss') AS wait_sec
    FROM    start_u s
    JOIN    end_u e
    ON      s.ds = e.ds
    AND     s.userid = e.userid
    AND     s.biz_code = e.biz_code
    WHERE   DATEDIFF(e.end_time, s.start_time, 'ss') BETWEEN 1 AND 7200
)
, active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260801'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, start_by AS (
    SELECT  s.ds
            ,a.赚取分档
            ,COUNT(1) AS 开始轮次
            ,COUNT(DISTINCT s.userid) AS 开始人数
    FROM    start_u s
    JOIN    active_w a
    ON      s.ds = a.ds
    AND     s.userid = a.userid
    GROUP BY s.ds
             ,a.赚取分档
)
, code_by AS (
    SELECT  s.ds
            ,a.赚取分档
            ,s.end_code
            ,COUNT(1) AS 配对轮次
            ,COUNT(DISTINCT s.userid) AS 人数
            ,ROUND(AVG(s.wait_sec), 2) AS 等待均值秒
            ,ROUND(PERCENTILE_APPROX(CAST(s.wait_sec AS DOUBLE), 0.5), 2) AS 等待中位数秒
    FROM    sess s
    JOIN    active_w a
    ON      s.ds = a.ds
    AND     s.userid = a.userid
    GROUP BY s.ds
             ,a.赚取分档
             ,s.end_code
)
, pair_tot AS (
    SELECT  ds
            ,赚取分档
            ,SUM(配对轮次) AS 配对轮次合计
    FROM    code_by
    GROUP BY ds
             ,赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    active_w
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档, 1 AS end_code
    UNION ALL SELECT 1, '50-女', 2
    UNION ALL SELECT 1, '50-女', 3
    UNION ALL SELECT 1, '50-100女', 1
    UNION ALL SELECT 1, '50-100女', 2
    UNION ALL SELECT 1, '50-100女', 3
    UNION ALL SELECT 1, '100+女', 1
    UNION ALL SELECT 1, '100+女', 2
    UNION ALL SELECT 1, '100+女', 3
    UNION ALL SELECT 1, '财富缺失', 1
    UNION ALL SELECT 1, '财富缺失', 2
    UNION ALL SELECT 1, '财富缺失', 3
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
            ,b.end_code
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,g.end_code AS `结束码`
        ,CASE g.end_code
             WHEN 1 THEN '1-成功接通结束'
             WHEN 2 THEN '2-主动挂断'
             WHEN 3 THEN '3-心跳超时'
         END AS `结束原因`
        ,AVG(NVL(b.开始轮次, 0)) AS `日均坐等轮次`
        ,AVG(NVL(b.开始人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(c.配对轮次, 0)) AS `日均该结束码轮次`
        ,AVG(NVL(c.人数, 0)) AS `日均该结束码人数`
        ,AVG(NVL(p.配对轮次合计, 0)) AS `日均配对轮次合计`
        ,AVG(CASE WHEN NVL(b.开始轮次, 0) = 0 THEN 0
                  ELSE ROUND(NVL(p.配对轮次合计, 0) * 1.0 / b.开始轮次, 4)
             END) AS `日均配对成功率`
        ,AVG(CASE WHEN NVL(p.配对轮次合计, 0) = 0 THEN 0
                  ELSE ROUND(NVL(c.配对轮次, 0) * 1.0 / p.配对轮次合计, 4)
             END) AS `日均占配对轮次`
        ,AVG(NVL(c.等待均值秒, 0)) AS `日均等待均值秒`
        ,AVG(NVL(c.等待中位数秒, 0)) AS `日均等待中位数秒`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN start_by b
ON      g.ds = b.ds
AND     g.赚取分档 = b.赚取分档
LEFT JOIN pair_tot p
ON      g.ds = p.ds
AND     g.赚取分档 = p.赚取分档
LEFT JOIN code_by c
ON      g.ds = c.ds
AND     g.赚取分档 = c.赚取分档
AND     g.end_code = c.end_code
GROUP BY g.赚取分档
         ,g.end_code
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
         ,g.end_code
;
```

### 7.3 下半月日均（20260802～20260817）

与 7.2 同一套 SQL，只改日期。本段 16 天。跑完后和 7.2 按天数加权合成近一月日均。

```sql
--下半月日均(20260802~20260817)：分档 × 结束码 + 坐等时长（333 先配对再 JOIN 分档）
WITH ev AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
            ,GET_JSON_OBJECT(s.extra, '$.bizCode') AS biz_code
            ,CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) AS wait_type
            ,CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) AS end_code
            ,TO_DATE(GET_JSON_OBJECT(s.extra, '$.createTime'), 'yyyy-mm-dd hh:mi:ss') AS evt_time
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260802' AND '20260817'
    AND     s.log_id = '333'
    AND     GET_JSON_OBJECT(s.extra, '$.bizCode') IS NOT NULL
    AND     (
                CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
                OR (
                    CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 2
                    AND CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) IN (1, 2, 3)
                )
            )
)
, start_u AS (
    SELECT  ds
            ,userid
            ,biz_code
            ,MIN(evt_time) AS start_time
    FROM    ev
    WHERE   wait_type = 1
    GROUP BY ds
             ,userid
             ,biz_code
)
, end_u AS (
    SELECT  ds
            ,userid
            ,biz_code
            ,end_time
            ,end_code
    FROM    (
                SELECT  ds
                        ,userid
                        ,biz_code
                        ,evt_time AS end_time
                        ,end_code
                        ,ROW_NUMBER() OVER (PARTITION BY ds, userid, biz_code ORDER BY evt_time DESC) AS rn
                FROM    ev
                WHERE   wait_type = 2
            ) t
    WHERE   rn = 1
)
, sess AS (
    SELECT  s.ds
            ,s.userid
            ,e.end_code
            ,DATEDIFF(e.end_time, s.start_time, 'ss') AS wait_sec
    FROM    start_u s
    JOIN    end_u e
    ON      s.ds = e.ds
    AND     s.userid = e.userid
    AND     s.biz_code = e.biz_code
    WHERE   DATEDIFF(e.end_time, s.start_time, 'ss') BETWEEN 1 AND 7200
)
, active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260802' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, start_by AS (
    SELECT  s.ds
            ,a.赚取分档
            ,COUNT(1) AS 开始轮次
            ,COUNT(DISTINCT s.userid) AS 开始人数
    FROM    start_u s
    JOIN    active_w a
    ON      s.ds = a.ds
    AND     s.userid = a.userid
    GROUP BY s.ds
             ,a.赚取分档
)
, code_by AS (
    SELECT  s.ds
            ,a.赚取分档
            ,s.end_code
            ,COUNT(1) AS 配对轮次
            ,COUNT(DISTINCT s.userid) AS 人数
            ,ROUND(AVG(s.wait_sec), 2) AS 等待均值秒
            ,ROUND(PERCENTILE_APPROX(CAST(s.wait_sec AS DOUBLE), 0.5), 2) AS 等待中位数秒
    FROM    sess s
    JOIN    active_w a
    ON      s.ds = a.ds
    AND     s.userid = a.userid
    GROUP BY s.ds
             ,a.赚取分档
             ,s.end_code
)
, pair_tot AS (
    SELECT  ds
            ,赚取分档
            ,SUM(配对轮次) AS 配对轮次合计
    FROM    code_by
    GROUP BY ds
             ,赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    active_w
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档, 1 AS end_code
    UNION ALL SELECT 1, '50-女', 2
    UNION ALL SELECT 1, '50-女', 3
    UNION ALL SELECT 1, '50-100女', 1
    UNION ALL SELECT 1, '50-100女', 2
    UNION ALL SELECT 1, '50-100女', 3
    UNION ALL SELECT 1, '100+女', 1
    UNION ALL SELECT 1, '100+女', 2
    UNION ALL SELECT 1, '100+女', 3
    UNION ALL SELECT 1, '财富缺失', 1
    UNION ALL SELECT 1, '财富缺失', 2
    UNION ALL SELECT 1, '财富缺失', 3
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
            ,b.end_code
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,g.end_code AS `结束码`
        ,CASE g.end_code
             WHEN 1 THEN '1-成功接通结束'
             WHEN 2 THEN '2-主动挂断'
             WHEN 3 THEN '3-心跳超时'
         END AS `结束原因`
        ,AVG(NVL(b.开始轮次, 0)) AS `日均坐等轮次`
        ,AVG(NVL(b.开始人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(c.配对轮次, 0)) AS `日均该结束码轮次`
        ,AVG(NVL(c.人数, 0)) AS `日均该结束码人数`
        ,AVG(NVL(p.配对轮次合计, 0)) AS `日均配对轮次合计`
        ,AVG(CASE WHEN NVL(b.开始轮次, 0) = 0 THEN 0
                  ELSE ROUND(NVL(p.配对轮次合计, 0) * 1.0 / b.开始轮次, 4)
             END) AS `日均配对成功率`
        ,AVG(CASE WHEN NVL(p.配对轮次合计, 0) = 0 THEN 0
                  ELSE ROUND(NVL(c.配对轮次, 0) * 1.0 / p.配对轮次合计, 4)
             END) AS `日均占配对轮次`
        ,AVG(NVL(c.等待均值秒, 0)) AS `日均等待均值秒`
        ,AVG(NVL(c.等待中位数秒, 0)) AS `日均等待中位数秒`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN start_by b
ON      g.ds = b.ds
AND     g.赚取分档 = b.赚取分档
LEFT JOIN pair_tot p
ON      g.ds = p.ds
AND     g.赚取分档 = p.赚取分档
LEFT JOIN code_by c
ON      g.ds = c.ds
AND     g.赚取分档 = c.赚取分档
AND     g.end_code = c.end_code
GROUP BY g.赚取分档
         ,g.end_code
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
         ,g.end_code
;
```

> 近一月日均（每一列、每一档×结束码）：`(7.2日均 × 7.2统计天数 + 7.3日均 × 7.3统计天数) / (7.2统计天数 + 7.3统计天数)`。统计天数应约为 15+16=31。

---

## 8. 看人均坐等次数

问：空坐是少数人反复进，还是很多人各坐一次。次数用当天 `type=1` 轮次（同一 `bizCode` 去重）；合计等待只来自 §7 能配对上的轮次。弹窗一天最多 5 次，人均若远高于 5，入口不止弹窗。

### 8.1 当日（20260817）

```sql
--20260817当日：分档人均坐等轮次 + 当天合计等待（333 先配对再 JOIN 分档）
WITH ev AS (
    SELECT  CAST(s.userid AS BIGINT) AS userid
            ,GET_JSON_OBJECT(s.extra, '$.bizCode') AS biz_code
            ,CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) AS wait_type
            ,CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) AS end_code
            ,TO_DATE(GET_JSON_OBJECT(s.extra, '$.createTime'), 'yyyy-mm-dd hh:mi:ss') AS evt_time
    FROM    chamet_data.server_log_data s
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     GET_JSON_OBJECT(s.extra, '$.bizCode') IS NOT NULL
    AND     (
                CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
                OR (
                    CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 2
                    AND CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) IN (1, 2, 3)
                )
            )
)
, start_u AS (
    SELECT  userid
            ,biz_code
            ,MIN(evt_time) AS start_time
    FROM    ev
    WHERE   wait_type = 1
    GROUP BY userid
             ,biz_code
)
, end_u AS (
    SELECT  userid
            ,biz_code
            ,end_time
    FROM    (
                SELECT  userid
                        ,biz_code
                        ,evt_time AS end_time
                        ,ROW_NUMBER() OVER (PARTITION BY userid, biz_code ORDER BY evt_time DESC) AS rn
                FROM    ev
                WHERE   wait_type = 2
            ) t
    WHERE   rn = 1
)
, sess AS (
    SELECT  s.userid
            ,DATEDIFF(e.end_time, s.start_time, 'ss') AS wait_sec
    FROM    start_u s
    JOIN    end_u e
    ON      s.userid = e.userid
    AND     s.biz_code = e.biz_code
    WHERE   DATEDIFF(e.end_time, s.start_time, 'ss') BETWEEN 1 AND 7200
)
, active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
)
, user_stat AS (
    SELECT  s.userid
            ,s.赚取分档
            ,s.开始轮次
            ,NVL(p.配对轮次, 0) AS 配对轮次
            ,NVL(p.合计等待秒, 0) AS 合计等待秒
    FROM    (
                SELECT  st.userid
                        ,a.赚取分档
                        ,COUNT(1) AS 开始轮次
                FROM    start_u st
                JOIN    active_w a
                ON      st.userid = a.userid
                GROUP BY st.userid
                         ,a.赚取分档
            ) s
    LEFT JOIN (
                SELECT  se.userid
                        ,a.赚取分档
                        ,COUNT(1) AS 配对轮次
                        ,SUM(se.wait_sec) AS 合计等待秒
                FROM    sess se
                JOIN    active_w a
                ON      se.userid = a.userid
                GROUP BY se.userid
                         ,a.赚取分档
            ) p
    ON      s.userid = p.userid
    AND     s.赚取分档 = p.赚取分档
)
SELECT  赚取分档 AS `赚取分档`
        ,COUNT(1) AS `坐等人数`
        ,SUM(开始轮次) AS `坐等轮次合计`
        ,SUM(配对轮次) AS `配对轮次合计`
        ,ROUND(AVG(开始轮次), 2) AS `人均坐等轮次`
        ,ROUND(PERCENTILE_APPROX(CAST(开始轮次 AS DOUBLE), 0.5), 2) AS `坐等轮次中位数`
        ,ROUND(PERCENTILE_APPROX(CAST(开始轮次 AS DOUBLE), 0.9), 2) AS `坐等轮次90分位`
        ,ROUND(AVG(配对轮次), 2) AS `人均配对轮次`
        ,ROUND(AVG(合计等待秒), 2) AS `人均合计等待均值秒`
        ,ROUND(PERCENTILE_APPROX(CAST(合计等待秒 AS DOUBLE), 0.5), 2) AS `人均合计等待中位数秒`
FROM    user_stat
GROUP BY 赚取分档
ORDER BY CASE 赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

### 8.2 上半月日均（20260718～20260801）

整月拆两段，避免 `ODPS-0020071`。本段 15 天。

```sql
--上半月日均(20260718~20260801)：分档人均坐等轮次 + 当天合计等待（333 先配对再 JOIN 分档）
WITH ev AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
            ,GET_JSON_OBJECT(s.extra, '$.bizCode') AS biz_code
            ,CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) AS wait_type
            ,CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) AS end_code
            ,TO_DATE(GET_JSON_OBJECT(s.extra, '$.createTime'), 'yyyy-mm-dd hh:mi:ss') AS evt_time
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260801'
    AND     s.log_id = '333'
    AND     GET_JSON_OBJECT(s.extra, '$.bizCode') IS NOT NULL
    AND     (
                CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
                OR (
                    CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 2
                    AND CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) IN (1, 2, 3)
                )
            )
)
, start_u AS (
    SELECT  ds
            ,userid
            ,biz_code
            ,MIN(evt_time) AS start_time
    FROM    ev
    WHERE   wait_type = 1
    GROUP BY ds
             ,userid
             ,biz_code
)
, end_u AS (
    SELECT  ds
            ,userid
            ,biz_code
            ,end_time
    FROM    (
                SELECT  ds
                        ,userid
                        ,biz_code
                        ,evt_time AS end_time
                        ,ROW_NUMBER() OVER (PARTITION BY ds, userid, biz_code ORDER BY evt_time DESC) AS rn
                FROM    ev
                WHERE   wait_type = 2
            ) t
    WHERE   rn = 1
)
, sess AS (
    SELECT  s.ds
            ,s.userid
            ,DATEDIFF(e.end_time, s.start_time, 'ss') AS wait_sec
    FROM    start_u s
    JOIN    end_u e
    ON      s.ds = e.ds
    AND     s.userid = e.userid
    AND     s.biz_code = e.biz_code
    WHERE   DATEDIFF(e.end_time, s.start_time, 'ss') BETWEEN 1 AND 7200
)
, active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260801'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, user_stat AS (
    SELECT  s.ds
            ,s.userid
            ,s.赚取分档
            ,s.开始轮次
            ,NVL(p.配对轮次, 0) AS 配对轮次
            ,NVL(p.合计等待秒, 0) AS 合计等待秒
    FROM    (
                SELECT  st.ds
                        ,st.userid
                        ,a.赚取分档
                        ,COUNT(1) AS 开始轮次
                FROM    start_u st
                JOIN    active_w a
                ON      st.ds = a.ds
                AND     st.userid = a.userid
                GROUP BY st.ds
                         ,st.userid
                         ,a.赚取分档
            ) s
    LEFT JOIN (
                SELECT  se.ds
                        ,se.userid
                        ,a.赚取分档
                        ,COUNT(1) AS 配对轮次
                        ,SUM(se.wait_sec) AS 合计等待秒
                FROM    sess se
                JOIN    active_w a
                ON      se.ds = a.ds
                AND     se.userid = a.userid
                GROUP BY se.ds
                         ,se.userid
                         ,a.赚取分档
            ) p
    ON      s.ds = p.ds
    AND     s.userid = p.userid
    AND     s.赚取分档 = p.赚取分档
)
, daily AS (
    SELECT  ds
            ,赚取分档
            ,COUNT(1) AS 坐等人数
            ,SUM(开始轮次) AS 开始轮次合计
            ,SUM(配对轮次) AS 配对轮次合计
            ,ROUND(AVG(开始轮次), 2) AS 人均开始轮次
            ,ROUND(PERCENTILE_APPROX(CAST(开始轮次 AS DOUBLE), 0.5), 2) AS 开始轮次中位数
            ,ROUND(PERCENTILE_APPROX(CAST(开始轮次 AS DOUBLE), 0.9), 2) AS 开始轮次90分位
            ,ROUND(AVG(配对轮次), 2) AS 人均配对轮次
            ,ROUND(AVG(合计等待秒), 2) AS 人均合计等待均值秒
            ,ROUND(PERCENTILE_APPROX(CAST(合计等待秒 AS DOUBLE), 0.5), 2) AS 人均合计等待中位数秒
    FROM    user_stat
    GROUP BY ds
             ,赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    active_w
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档
    UNION ALL SELECT 1, '50-100女'
    UNION ALL SELECT 1, '100+女'
    UNION ALL SELECT 1, '财富缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.开始轮次合计, 0)) AS `日均坐等轮次合计`
        ,AVG(NVL(d.配对轮次合计, 0)) AS `日均配对轮次合计`
        ,AVG(NVL(d.人均开始轮次, 0)) AS `日均人均坐等轮次`
        ,AVG(NVL(d.开始轮次中位数, 0)) AS `日均坐等轮次中位数`
        ,AVG(NVL(d.开始轮次90分位, 0)) AS `日均坐等轮次90分位`
        ,AVG(NVL(d.人均配对轮次, 0)) AS `日均人均配对轮次`
        ,AVG(NVL(d.人均合计等待均值秒, 0)) AS `日均人均合计等待均值秒`
        ,AVG(NVL(d.人均合计等待中位数秒, 0)) AS `日均人均合计等待中位数秒`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.赚取分档 = d.赚取分档
GROUP BY g.赚取分档
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

### 8.3 下半月日均（20260802～20260817）

与 8.2 同一套 SQL，只改日期。本段 16 天。跑完后和 8.2 按天数加权合成近一月日均。

```sql
--下半月日均(20260802~20260817)：分档人均坐等轮次 + 当天合计等待（333 先配对再 JOIN 分档）
WITH ev AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
            ,GET_JSON_OBJECT(s.extra, '$.bizCode') AS biz_code
            ,CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) AS wait_type
            ,CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) AS end_code
            ,TO_DATE(GET_JSON_OBJECT(s.extra, '$.createTime'), 'yyyy-mm-dd hh:mi:ss') AS evt_time
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260802' AND '20260817'
    AND     s.log_id = '333'
    AND     GET_JSON_OBJECT(s.extra, '$.bizCode') IS NOT NULL
    AND     (
                CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
                OR (
                    CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 2
                    AND CAST(GET_JSON_OBJECT(s.extra, '$.endCode') AS BIGINT) IN (1, 2, 3)
                )
            )
)
, start_u AS (
    SELECT  ds
            ,userid
            ,biz_code
            ,MIN(evt_time) AS start_time
    FROM    ev
    WHERE   wait_type = 1
    GROUP BY ds
             ,userid
             ,biz_code
)
, end_u AS (
    SELECT  ds
            ,userid
            ,biz_code
            ,end_time
    FROM    (
                SELECT  ds
                        ,userid
                        ,biz_code
                        ,evt_time AS end_time
                        ,ROW_NUMBER() OVER (PARTITION BY ds, userid, biz_code ORDER BY evt_time DESC) AS rn
                FROM    ev
                WHERE   wait_type = 2
            ) t
    WHERE   rn = 1
)
, sess AS (
    SELECT  s.ds
            ,s.userid
            ,DATEDIFF(e.end_time, s.start_time, 'ss') AS wait_sec
    FROM    start_u s
    JOIN    end_u e
    ON      s.ds = e.ds
    AND     s.userid = e.userid
    AND     s.biz_code = e.biz_code
    WHERE   DATEDIFF(e.end_time, s.start_time, 'ss') BETWEEN 1 AND 7200
)
, active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260802' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, user_stat AS (
    SELECT  s.ds
            ,s.userid
            ,s.赚取分档
            ,s.开始轮次
            ,NVL(p.配对轮次, 0) AS 配对轮次
            ,NVL(p.合计等待秒, 0) AS 合计等待秒
    FROM    (
                SELECT  st.ds
                        ,st.userid
                        ,a.赚取分档
                        ,COUNT(1) AS 开始轮次
                FROM    start_u st
                JOIN    active_w a
                ON      st.ds = a.ds
                AND     st.userid = a.userid
                GROUP BY st.ds
                         ,st.userid
                         ,a.赚取分档
            ) s
    LEFT JOIN (
                SELECT  se.ds
                        ,se.userid
                        ,a.赚取分档
                        ,COUNT(1) AS 配对轮次
                        ,SUM(se.wait_sec) AS 合计等待秒
                FROM    sess se
                JOIN    active_w a
                ON      se.ds = a.ds
                AND     se.userid = a.userid
                GROUP BY se.ds
                         ,se.userid
                         ,a.赚取分档
            ) p
    ON      s.ds = p.ds
    AND     s.userid = p.userid
    AND     s.赚取分档 = p.赚取分档
)
, daily AS (
    SELECT  ds
            ,赚取分档
            ,COUNT(1) AS 坐等人数
            ,SUM(开始轮次) AS 开始轮次合计
            ,SUM(配对轮次) AS 配对轮次合计
            ,ROUND(AVG(开始轮次), 2) AS 人均开始轮次
            ,ROUND(PERCENTILE_APPROX(CAST(开始轮次 AS DOUBLE), 0.5), 2) AS 开始轮次中位数
            ,ROUND(PERCENTILE_APPROX(CAST(开始轮次 AS DOUBLE), 0.9), 2) AS 开始轮次90分位
            ,ROUND(AVG(配对轮次), 2) AS 人均配对轮次
            ,ROUND(AVG(合计等待秒), 2) AS 人均合计等待均值秒
            ,ROUND(PERCENTILE_APPROX(CAST(合计等待秒 AS DOUBLE), 0.5), 2) AS 人均合计等待中位数秒
    FROM    user_stat
    GROUP BY ds
             ,赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    active_w
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档
    UNION ALL SELECT 1, '50-100女'
    UNION ALL SELECT 1, '100+女'
    UNION ALL SELECT 1, '财富缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.开始轮次合计, 0)) AS `日均坐等轮次合计`
        ,AVG(NVL(d.配对轮次合计, 0)) AS `日均配对轮次合计`
        ,AVG(NVL(d.人均开始轮次, 0)) AS `日均人均坐等轮次`
        ,AVG(NVL(d.开始轮次中位数, 0)) AS `日均坐等轮次中位数`
        ,AVG(NVL(d.开始轮次90分位, 0)) AS `日均坐等轮次90分位`
        ,AVG(NVL(d.人均配对轮次, 0)) AS `日均人均配对轮次`
        ,AVG(NVL(d.人均合计等待均值秒, 0)) AS `日均人均合计等待均值秒`
        ,AVG(NVL(d.人均合计等待中位数秒, 0)) AS `日均人均合计等待中位数秒`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.赚取分档 = d.赚取分档
GROUP BY g.赚取分档
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

> 近一月日均（每一列、每一档）：`(8.2日均 × 8.2统计天数 + 8.3日均 × 8.3统计天数) / (8.2统计天数 + 8.3统计天数)`。统计天数应约为 15+16=31。

---

## 9. 看坐等接通单厚度

问：已经接通的人，是通数少，还是通了也短、也便宜。分母是坐等接通女。通话时长用订单 `call_time`（没有则 `valid_time`），均值+中位数都出。金豆仍只计 `receive_type=12` 接通单。

### 9.1 当日（20260817）

```sql
--20260817当日：坐等接通订单厚度（人均通数 / 场均时长 / 场均金豆，均值+中位数）
WITH active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  DISTINCT CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    JOIN    active_w a
    ON      CAST(s.userid AS BIGINT) = a.userid
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
)
, conn_ord AS (
    SELECT  v.called_id AS userid
            ,a.赚取分档
            ,CASE WHEN NVL(v.call_time, 0) > 0 THEN v.call_time ELSE v.valid_time END AS 通话秒
            ,NVL(v.integral, 0) + NVL(v.integral_give, 0) + NVL(v.gift_integral, 0) AS 金豆
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    JOIN    active_w a
    ON      v.called_id = a.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
)
SELECT  赚取分档 AS `赚取分档`
        ,COUNT(DISTINCT userid) AS `坐等接通人数`
        ,COUNT(1) AS `接通订单数`
        ,ROUND(COUNT(1) * 1.0 / COUNT(DISTINCT userid), 2) AS `人均通数`
        ,ROUND(AVG(通话秒), 2) AS `场均通话均值秒`
        ,ROUND(PERCENTILE_APPROX(CAST(通话秒 AS DOUBLE), 0.5), 2) AS `场均通话中位数秒`
        ,ROUND(AVG(金豆), 2) AS `场均金豆均值`
        ,ROUND(PERCENTILE_APPROX(CAST(金豆 AS DOUBLE), 0.5), 2) AS `场均金豆中位数`
        ,SUM(金豆) AS `接通金豆合计`
        ,ROUND(SUM(金豆) * 1.0 / COUNT(DISTINCT userid), 2) AS `接通人均金豆`
FROM    conn_ord
GROUP BY 赚取分档
ORDER BY CASE 赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

### 9.2 近一月日均（20260718～20260817）

```sql
--近一月日均(20260718~20260817)：坐等接通订单厚度
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, conn_ord AS (
    SELECT  v.ds
            ,v.called_id AS userid
            ,a.赚取分档
            ,CASE WHEN NVL(v.call_time, 0) > 0 THEN v.call_time ELSE v.valid_time END AS 通话秒
            ,NVL(v.integral, 0) + NVL(v.integral_give, 0) + NVL(v.gift_integral, 0) AS 金豆
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    JOIN    active_w a
    ON      v.ds = a.ds
    AND     v.called_id = a.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
)
, daily AS (
    SELECT  ds
            ,赚取分档
            ,COUNT(DISTINCT userid) AS 接通人数
            ,COUNT(1) AS 接通订单数
            ,ROUND(COUNT(1) * 1.0 / COUNT(DISTINCT userid), 2) AS 人均通数
            ,ROUND(AVG(通话秒), 2) AS 场均通话均值秒
            ,ROUND(PERCENTILE_APPROX(CAST(通话秒 AS DOUBLE), 0.5), 2) AS 场均通话中位数秒
            ,ROUND(AVG(金豆), 2) AS 场均金豆均值
            ,ROUND(PERCENTILE_APPROX(CAST(金豆 AS DOUBLE), 0.5), 2) AS 场均金豆中位数
            ,SUM(金豆) AS 接通金豆合计
            ,ROUND(SUM(金豆) * 1.0 / COUNT(DISTINCT userid), 2) AS 接通人均金豆
    FROM    conn_ord
    GROUP BY ds
             ,赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    active_w
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档
    UNION ALL SELECT 1, '50-100女'
    UNION ALL SELECT 1, '100+女'
    UNION ALL SELECT 1, '财富缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,AVG(NVL(d.接通人数, 0)) AS `日均坐等接通人数`
        ,AVG(NVL(d.接通订单数, 0)) AS `日均接通订单数`
        ,AVG(NVL(d.人均通数, 0)) AS `日均人均通数`
        ,AVG(NVL(d.场均通话均值秒, 0)) AS `日均场均通话均值秒`
        ,AVG(NVL(d.场均通话中位数秒, 0)) AS `日均场均通话中位数秒`
        ,AVG(NVL(d.场均金豆均值, 0)) AS `日均场均金豆均值`
        ,AVG(NVL(d.场均金豆中位数, 0)) AS `日均场均金豆中位数`
        ,AVG(NVL(d.接通金豆合计, 0)) AS `日均接通金豆合计`
        ,AVG(NVL(d.接通人均金豆, 0)) AS `日均接通人均金豆`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.赚取分档 = d.赚取分档
GROUP BY g.赚取分档
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

---

## 10. 看坐等女当天其他视频聊

问：空坐是不是当天唯一被叫路径。分母是坐等女。坐等被叫 = `receive_type=12`；其他被叫 = 男打女且 `receive_type≠12`。两种都无 = 坐等了但当天没有任何男打女订单。

### 10.1 当日（20260817）

```sql
--20260817当日：坐等女 × 坐等被叫 / 其他被叫四格
WITH active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  DISTINCT CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    JOIN    active_w a
    ON      CAST(s.userid AS BIGINT) = a.userid
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
)
, sit_hit AS (
    SELECT  v.called_id AS userid
            ,MAX(CASE WHEN NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0 THEN 1 ELSE 0 END) AS 有接通
            ,SUM(CASE WHEN NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0
                      THEN NVL(v.integral, 0) + NVL(v.integral_give, 0) + NVL(v.gift_integral, 0)
                      ELSE 0 END) AS 接通金豆
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    GROUP BY v.called_id
)
, oth_hit AS (
    SELECT  v.called_id AS userid
            ,MAX(CASE WHEN NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0 THEN 1 ELSE 0 END) AS 有接通
            ,SUM(CASE WHEN NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0
                      THEN NVL(v.integral, 0) + NVL(v.integral_give, 0) + NVL(v.gift_integral, 0)
                      ELSE 0 END) AS 接通金豆
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     (v.receive_type IS NULL OR v.receive_type <> 12)
    GROUP BY v.called_id
)
SELECT  a.赚取分档 AS `赚取分档`
        ,COUNT(1) AS `坐等人数`
        ,SUM(CASE WHEN s.userid IS NOT NULL AND o.userid IS NULL THEN 1 ELSE 0 END) AS `只坐等被叫人数`
        ,SUM(CASE WHEN s.userid IS NULL AND o.userid IS NOT NULL THEN 1 ELSE 0 END) AS `只其他被叫人数`
        ,SUM(CASE WHEN s.userid IS NOT NULL AND o.userid IS NOT NULL THEN 1 ELSE 0 END) AS `两种都有人数`
        ,SUM(CASE WHEN s.userid IS NULL AND o.userid IS NULL THEN 1 ELSE 0 END) AS `两种都无人数`
        ,ROUND(SUM(CASE WHEN s.userid IS NULL AND o.userid IS NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(1), 4) AS `两种都无占比`
        ,SUM(CASE WHEN NVL(s.有接通, 0) = 1 THEN 1 ELSE 0 END) AS `坐等接通人数`
        ,SUM(CASE WHEN NVL(o.有接通, 0) = 1 THEN 1 ELSE 0 END) AS `其他接通人数`
        ,SUM(NVL(s.接通金豆, 0)) AS `坐等接通金豆合计`
        ,SUM(NVL(o.接通金豆, 0)) AS `其他接通金豆合计`
FROM    wait_u w
JOIN    active_w a
ON      w.userid = a.userid
LEFT JOIN sit_hit s
ON      w.userid = s.userid
LEFT JOIN oth_hit o
ON      w.userid = o.userid
GROUP BY a.赚取分档
ORDER BY CASE a.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

### 10.2 近一月日均（20260718～20260817）

```sql
--近一月日均(20260718~20260817)：坐等女 × 坐等被叫 / 其他被叫四格
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.history_earn IS NULL THEN '财富缺失'
                WHEN u.history_earn < 50000 THEN '50-女'
                WHEN u.history_earn < 100000 THEN '50-100女'
                ELSE '100+女'
             END AS 赚取分档
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, sit_hit AS (
    SELECT  v.ds
            ,v.called_id AS userid
            ,MAX(CASE WHEN NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0 THEN 1 ELSE 0 END) AS 有接通
            ,SUM(CASE WHEN NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0
                      THEN NVL(v.integral, 0) + NVL(v.integral_give, 0) + NVL(v.gift_integral, 0)
                      ELSE 0 END) AS 接通金豆
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    GROUP BY v.ds, v.called_id
)
, oth_hit AS (
    SELECT  v.ds
            ,v.called_id AS userid
            ,MAX(CASE WHEN NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0 THEN 1 ELSE 0 END) AS 有接通
            ,SUM(CASE WHEN NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0
                      THEN NVL(v.integral, 0) + NVL(v.integral_give, 0) + NVL(v.gift_integral, 0)
                      ELSE 0 END) AS 接通金豆
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     (v.receive_type IS NULL OR v.receive_type <> 12)
    GROUP BY v.ds, v.called_id
)
, daily AS (
    SELECT  a.ds
            ,a.赚取分档
            ,COUNT(1) AS 坐等人数
            ,SUM(CASE WHEN s.userid IS NOT NULL AND o.userid IS NULL THEN 1 ELSE 0 END) AS 只坐等被叫人数
            ,SUM(CASE WHEN s.userid IS NULL AND o.userid IS NOT NULL THEN 1 ELSE 0 END) AS 只其他被叫人数
            ,SUM(CASE WHEN s.userid IS NOT NULL AND o.userid IS NOT NULL THEN 1 ELSE 0 END) AS 两种都有人数
            ,SUM(CASE WHEN s.userid IS NULL AND o.userid IS NULL THEN 1 ELSE 0 END) AS 两种都无人数
            ,CASE WHEN COUNT(1) = 0 THEN 0
                  ELSE ROUND(SUM(CASE WHEN s.userid IS NULL AND o.userid IS NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(1), 4)
             END AS 两种都无占比
            ,SUM(CASE WHEN NVL(s.有接通, 0) = 1 THEN 1 ELSE 0 END) AS 坐等接通人数
            ,SUM(CASE WHEN NVL(o.有接通, 0) = 1 THEN 1 ELSE 0 END) AS 其他接通人数
            ,SUM(NVL(s.接通金豆, 0)) AS 坐等接通金豆合计
            ,SUM(NVL(o.接通金豆, 0)) AS 其他接通金豆合计
    FROM    wait_u w
    JOIN    active_w a
    ON      w.ds = a.ds
    AND     w.userid = a.userid
    LEFT JOIN sit_hit s
    ON      w.ds = s.ds
    AND     w.userid = s.userid
    LEFT JOIN oth_hit o
    ON      w.ds = o.ds
    AND     w.userid = o.userid
    GROUP BY a.ds
             ,a.赚取分档
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    active_w
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, '50-女' AS 赚取分档
    UNION ALL SELECT 1, '50-100女'
    UNION ALL SELECT 1, '100+女'
    UNION ALL SELECT 1, '财富缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.赚取分档
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.赚取分档 AS `赚取分档`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.只坐等被叫人数, 0)) AS `日均只坐等被叫人数`
        ,AVG(NVL(d.只其他被叫人数, 0)) AS `日均只其他被叫人数`
        ,AVG(NVL(d.两种都有人数, 0)) AS `日均两种都有人数`
        ,AVG(NVL(d.两种都无人数, 0)) AS `日均两种都无人数`
        ,AVG(NVL(d.两种都无占比, 0)) AS `日均两种都无占比`
        ,AVG(NVL(d.坐等接通人数, 0)) AS `日均坐等接通人数`
        ,AVG(NVL(d.其他接通人数, 0)) AS `日均其他接通人数`
        ,AVG(NVL(d.坐等接通金豆合计, 0)) AS `日均坐等接通金豆合计`
        ,AVG(NVL(d.其他接通金豆合计, 0)) AS `日均其他接通金豆合计`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.赚取分档 = d.赚取分档
GROUP BY g.赚取分档
ORDER BY CASE g.赚取分档
             WHEN '50-女' THEN 1
             WHEN '50-100女' THEN 2
             WHEN '100+女' THEN 3
             ELSE 4
         END
;
```

---

## 11. 看 50- 注册后天数

问：空坐是注册当天特有，还是过了首日还在空。只拆 50-。D0=注册当天，D1-7，D8+。指标沿用坐等率 / 接通率 / 开播率 / 只坐等不开播。

### 11.1 当日（20260817）

```sql
--20260817当日：50-女 × 注册后天数（D0 / D1-7 / D8+）
WITH active_w AS (
    SELECT  a.userid
            ,CASE
                WHEN u.create_time IS NULL
                     OR LENGTH(SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8)) < 8
                THEN '注册日缺失'
                WHEN DATEDIFF(
                         TO_DATE('20260817', 'yyyymmdd')
                        ,TO_DATE(SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8), 'yyyymmdd')
                        ,'dd'
                     ) = 0 THEN 'D0-注册当天'
                WHEN DATEDIFF(
                         TO_DATE('20260817', 'yyyymmdd')
                        ,TO_DATE(SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8), 'yyyymmdd')
                        ,'dd'
                     ) BETWEEN 1 AND 7 THEN 'D1-7'
                ELSE 'D8+'
             END AS 注册后天数
    FROM    (
                SELECT  userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds = '20260817'
                GROUP BY userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = '20260817'
    AND     u.sex = 0
    AND     NVL(u.history_earn, 0) < 50000
)
, wait_u AS (
    SELECT  DISTINCT CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    JOIN    active_w a
    ON      CAST(s.userid AS BIGINT) = a.userid
    WHERE   s.ds = '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
)
, conn_u AS (
    SELECT  DISTINCT v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
)
, hit_u AS (
    SELECT  DISTINCT v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.called_id = w.userid
    WHERE   v.ds = '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
)
, live_u AS (
    SELECT  DISTINCT b.owner_id AS userid
    FROM    chamet_data.t_room_statistic a
    JOIN    chamet_data.t_room b
    ON      a.room_id = b.room_id
    AND     b.type = 2
    JOIN    active_w w
    ON      b.owner_id = w.userid
    WHERE   a.ds = '20260817'
)
SELECT  a.注册后天数 AS `注册后天数`
        ,COUNT(1) AS `活跃女人数`
        ,COUNT(w.userid) AS `当天坐等人数`
        ,COUNT(h.userid) AS `坐等被打到人数`
        ,COUNT(c.userid) AS `坐等接通人数`
        ,COUNT(l.userid) AS `当天开播人数`
        ,ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4) AS `当天坐等率`
        ,CASE WHEN COUNT(w.userid) = 0 THEN 0
              ELSE ROUND(COUNT(h.userid) * 1.0 / COUNT(w.userid), 4)
         END AS `坐等到被打到`
        ,CASE WHEN COUNT(w.userid) = 0 THEN 0
              ELSE ROUND(COUNT(c.userid) * 1.0 / COUNT(w.userid), 4)
         END AS `坐等接通率`
        ,ROUND(COUNT(l.userid) * 1.0 / COUNT(1), 4) AS `当天开播率`
        ,ROUND(SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(1), 4) AS `只坐等不开播占比`
FROM    active_w a
LEFT JOIN wait_u w ON a.userid = w.userid
LEFT JOIN hit_u h ON a.userid = h.userid
LEFT JOIN conn_u c ON a.userid = c.userid
LEFT JOIN live_u l ON a.userid = l.userid
GROUP BY a.注册后天数
ORDER BY CASE a.注册后天数
             WHEN 'D0-注册当天' THEN 1
             WHEN 'D1-7' THEN 2
             WHEN 'D8+' THEN 3
             ELSE 4
         END
;
```

### 11.2 上半月日均（20260718～20260801）

整月拆两段，避免 `ODPS-0020071`。本段 15 天。

```sql
--上半月日均(20260718~20260801)：50-女 × 注册后天数（D0 / D1-7 / D8+）
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.create_time IS NULL
                     OR LENGTH(SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8)) < 8
                THEN '注册日缺失'
                WHEN DATEDIFF(
                         TO_DATE(a.ds, 'yyyymmdd')
                        ,TO_DATE(SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8), 'yyyymmdd')
                        ,'dd'
                     ) = 0 THEN 'D0-注册当天'
                WHEN DATEDIFF(
                         TO_DATE(a.ds, 'yyyymmdd')
                        ,TO_DATE(SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8), 'yyyymmdd')
                        ,'dd'
                     ) BETWEEN 1 AND 7 THEN 'D1-7'
                ELSE 'D8+'
             END AS 注册后天数
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260718' AND '20260801'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
    AND     NVL(u.history_earn, 0) < 50000
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260718' AND '20260801'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, conn_u AS (
    SELECT  v.ds
            ,v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260801'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
    GROUP BY v.ds, v.called_id
)
, hit_u AS (
    SELECT  v.ds
            ,v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260718' AND '20260801'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    GROUP BY v.ds, v.called_id
)
, live_u AS (
    SELECT  a.ds
            ,b.owner_id AS userid
    FROM    chamet_data.t_room_statistic a
    JOIN    chamet_data.t_room b
    ON      a.room_id = b.room_id
    AND     b.type = 2
    JOIN    active_w w
    ON      a.ds = w.ds
    AND     b.owner_id = w.userid
    WHERE   a.ds BETWEEN '20260718' AND '20260801'
    GROUP BY a.ds, b.owner_id
)
, daily AS (
    SELECT  a.ds
            ,a.注册后天数
            ,COUNT(1) AS 活跃女人数
            ,COUNT(w.userid) AS 坐等人数
            ,COUNT(h.userid) AS 被打到人数
            ,COUNT(c.userid) AS 接通人数
            ,COUNT(l.userid) AS 开播人数
            ,CASE WHEN COUNT(1) = 0 THEN 0
                  ELSE ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4)
             END AS 坐等率
            ,CASE WHEN COUNT(w.userid) = 0 THEN 0
                  ELSE ROUND(COUNT(h.userid) * 1.0 / COUNT(w.userid), 4)
             END AS 坐等到被打到
            ,CASE WHEN COUNT(w.userid) = 0 THEN 0
                  ELSE ROUND(COUNT(c.userid) * 1.0 / COUNT(w.userid), 4)
             END AS 坐等接通率
            ,CASE WHEN COUNT(1) = 0 THEN 0
                  ELSE ROUND(COUNT(l.userid) * 1.0 / COUNT(1), 4)
             END AS 开播率
            ,CASE WHEN COUNT(1) = 0 THEN 0
                  ELSE ROUND(SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(1), 4)
             END AS 只坐等不开播占比
    FROM    active_w a
    LEFT JOIN wait_u w
    ON      a.ds = w.ds
    AND     a.userid = w.userid
    LEFT JOIN hit_u h
    ON      a.ds = h.ds
    AND     a.userid = h.userid
    LEFT JOIN conn_u c
    ON      a.ds = c.ds
    AND     a.userid = c.userid
    LEFT JOIN live_u l
    ON      a.ds = l.ds
    AND     a.userid = l.userid
    GROUP BY a.ds
             ,a.注册后天数
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    active_w
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, 'D0-注册当天' AS 注册后天数
    UNION ALL SELECT 1, 'D1-7'
    UNION ALL SELECT 1, 'D8+'
    UNION ALL SELECT 1, '注册日缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.注册后天数
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.注册后天数 AS `注册后天数`
        ,AVG(NVL(d.活跃女人数, 0)) AS `日均活跃女人数`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.被打到人数, 0)) AS `日均坐等被打到人数`
        ,AVG(NVL(d.接通人数, 0)) AS `日均坐等接通人数`
        ,AVG(NVL(d.开播人数, 0)) AS `日均开播人数`
        ,AVG(NVL(d.坐等率, 0)) AS `日均坐等率`
        ,AVG(NVL(d.坐等到被打到, 0)) AS `日均坐等到被打到`
        ,AVG(NVL(d.坐等接通率, 0)) AS `日均坐等接通率`
        ,AVG(NVL(d.开播率, 0)) AS `日均开播率`
        ,AVG(NVL(d.只坐等不开播占比, 0)) AS `日均只坐等不开播占比`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.注册后天数 = d.注册后天数
GROUP BY g.注册后天数
ORDER BY CASE g.注册后天数
             WHEN 'D0-注册当天' THEN 1
             WHEN 'D1-7' THEN 2
             WHEN 'D8+' THEN 3
             ELSE 4
         END
;
```

### 11.3 下半月日均（20260802～20260817）

与 11.2 同一套 SQL，只改日期。本段 16 天。跑完后和 11.2 按天数加权合成近一月日均。

```sql
--下半月日均(20260802~20260817)：50-女 × 注册后天数（D0 / D1-7 / D8+）
WITH active_w AS (
    SELECT  a.ds
            ,a.userid
            ,CASE
                WHEN u.create_time IS NULL
                     OR LENGTH(SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8)) < 8
                THEN '注册日缺失'
                WHEN DATEDIFF(
                         TO_DATE(a.ds, 'yyyymmdd')
                        ,TO_DATE(SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8), 'yyyymmdd')
                        ,'dd'
                     ) = 0 THEN 'D0-注册当天'
                WHEN DATEDIFF(
                         TO_DATE(a.ds, 'yyyymmdd')
                        ,TO_DATE(SUBSTR(REGEXP_REPLACE(u.create_time, '[^0-9]', ''), 1, 8), 'yyyymmdd')
                        ,'dd'
                     ) BETWEEN 1 AND 7 THEN 'D1-7'
                ELSE 'D8+'
             END AS 注册后天数
    FROM    (
                SELECT  ds
                        ,userid
                FROM    chamet_data.t_userlog_retained
                WHERE   ds BETWEEN '20260802' AND '20260817'
                GROUP BY ds, userid
            ) a
    JOIN    chamet_data.dws_user_f_d u
    ON      a.userid = u.userid
    AND     u.ds = a.ds
    AND     u.sex = 0
    AND     NVL(u.history_earn, 0) < 50000
)
, wait_u AS (
    SELECT  s.ds
            ,CAST(s.userid AS BIGINT) AS userid
    FROM    chamet_data.server_log_data s
    WHERE   s.ds BETWEEN '20260802' AND '20260817'
    AND     s.log_id = '333'
    AND     CAST(GET_JSON_OBJECT(s.extra, '$.type') AS BIGINT) = 1
    GROUP BY s.ds, CAST(s.userid AS BIGINT)
)
, conn_u AS (
    SELECT  v.ds
            ,v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260802' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    AND     (NVL(v.call_time, 0) > 0 OR NVL(v.valid_time, 0) > 0)
    GROUP BY v.ds, v.called_id
)
, hit_u AS (
    SELECT  v.ds
            ,v.called_id AS userid
    FROM    chamet_data.dwd_video_detail_i_d v
    JOIN    wait_u w
    ON      v.ds = w.ds
    AND     v.called_id = w.userid
    WHERE   v.ds BETWEEN '20260802' AND '20260817'
    AND     v.called_sex = 0
    AND     v.caller_sex = 1
    AND     v.receive_type = 12
    GROUP BY v.ds, v.called_id
)
, live_u AS (
    SELECT  a.ds
            ,b.owner_id AS userid
    FROM    chamet_data.t_room_statistic a
    JOIN    chamet_data.t_room b
    ON      a.room_id = b.room_id
    AND     b.type = 2
    JOIN    active_w w
    ON      a.ds = w.ds
    AND     b.owner_id = w.userid
    WHERE   a.ds BETWEEN '20260802' AND '20260817'
    GROUP BY a.ds, b.owner_id
)
, daily AS (
    SELECT  a.ds
            ,a.注册后天数
            ,COUNT(1) AS 活跃女人数
            ,COUNT(w.userid) AS 坐等人数
            ,COUNT(h.userid) AS 被打到人数
            ,COUNT(c.userid) AS 接通人数
            ,COUNT(l.userid) AS 开播人数
            ,CASE WHEN COUNT(1) = 0 THEN 0
                  ELSE ROUND(COUNT(w.userid) * 1.0 / COUNT(1), 4)
             END AS 坐等率
            ,CASE WHEN COUNT(w.userid) = 0 THEN 0
                  ELSE ROUND(COUNT(h.userid) * 1.0 / COUNT(w.userid), 4)
             END AS 坐等到被打到
            ,CASE WHEN COUNT(w.userid) = 0 THEN 0
                  ELSE ROUND(COUNT(c.userid) * 1.0 / COUNT(w.userid), 4)
             END AS 坐等接通率
            ,CASE WHEN COUNT(1) = 0 THEN 0
                  ELSE ROUND(COUNT(l.userid) * 1.0 / COUNT(1), 4)
             END AS 开播率
            ,CASE WHEN COUNT(1) = 0 THEN 0
                  ELSE ROUND(SUM(CASE WHEN w.userid IS NOT NULL AND l.userid IS NULL THEN 1 ELSE 0 END) * 1.0 / COUNT(1), 4)
             END AS 只坐等不开播占比
    FROM    active_w a
    LEFT JOIN wait_u w
    ON      a.ds = w.ds
    AND     a.userid = w.userid
    LEFT JOIN hit_u h
    ON      a.ds = h.ds
    AND     a.userid = h.userid
    LEFT JOIN conn_u c
    ON      a.ds = c.ds
    AND     a.userid = c.userid
    LEFT JOIN live_u l
    ON      a.ds = l.ds
    AND     a.userid = l.userid
    GROUP BY a.ds
             ,a.注册后天数
)
, calendar AS (
    SELECT  1 AS k
            ,ds
    FROM    active_w
    GROUP BY ds
)
, bucket_dim AS (
    SELECT  1 AS k, 'D0-注册当天' AS 注册后天数
    UNION ALL SELECT 1, 'D1-7'
    UNION ALL SELECT 1, 'D8+'
    UNION ALL SELECT 1, '注册日缺失'
)
, grid AS (
    SELECT  /*+ MAPJOIN(b) */
            c.ds
            ,b.注册后天数
    FROM    calendar c
    JOIN    bucket_dim b
    ON      c.k = b.k
)
SELECT  g.注册后天数 AS `注册后天数`
        ,AVG(NVL(d.活跃女人数, 0)) AS `日均活跃女人数`
        ,AVG(NVL(d.坐等人数, 0)) AS `日均坐等人数`
        ,AVG(NVL(d.被打到人数, 0)) AS `日均坐等被打到人数`
        ,AVG(NVL(d.接通人数, 0)) AS `日均坐等接通人数`
        ,AVG(NVL(d.开播人数, 0)) AS `日均开播人数`
        ,AVG(NVL(d.坐等率, 0)) AS `日均坐等率`
        ,AVG(NVL(d.坐等到被打到, 0)) AS `日均坐等到被打到`
        ,AVG(NVL(d.坐等接通率, 0)) AS `日均坐等接通率`
        ,AVG(NVL(d.开播率, 0)) AS `日均开播率`
        ,AVG(NVL(d.只坐等不开播占比, 0)) AS `日均只坐等不开播占比`
        ,COUNT(1) AS `统计天数`
FROM    grid g
LEFT JOIN daily d
ON      g.ds = d.ds
AND     g.注册后天数 = d.注册后天数
GROUP BY g.注册后天数
ORDER BY CASE g.注册后天数
             WHEN 'D0-注册当天' THEN 1
             WHEN 'D1-7' THEN 2
             WHEN 'D8+' THEN 3
             ELSE 4
         END
;
```

> 近一月日均（每一列、每一注册后天数档）：`(11.2日均 × 11.2统计天数 + 11.3日均 × 11.3统计天数) / (11.2统计天数 + 11.3统计天数)`。统计天数应约为 15+16=31。

---

## 跑数顺序

1. 先跑 §0，确认 `type=1` 仍是开始坐等。  
2. §1 对人数：50- 活跃女应明显大于注册当天女。  
3. §2 / §3 漏斗：接通 ≤ 被打到 ≤ 坐等 ≤ 活跃。  
4. 要讨论藏匹配看 §4，对原表看 §5，看代价看 §6。  
5. 空坐轮次看 §7（结束码+时长，均值和中位数都看）/ §8（人均次数）；接通厚度看 §9；是不是唯一接聊看 §10；50- 生命周期看 §11。  
6. 时长用 `extra.createTime`、同一 `bizCode` 在同一 `ds` 内配对；跨天对不上。先用 0817 对 `userid=175927843` 那条 48 秒。  
7. 埋点表一律 `server_log_data`（按 `ds` 分区），`WHERE` 必带 `ds`+`log_id='333'`。  
8. §7.2/7.3、§8.2/8.3、§11.2/11.3 拆成上半月（15 天）+ 下半月（16 天）。整月日均按两段 `统计天数` 加权，不要直接相加。
