# ElasticSearch技巧

## 查看集群节点的磁盘状态

http://127.0.0.1:9200/_cat/allocation?v&pretty

## ES滚动方式获取更多数据

select /*! USE_SCROLL(10,600000)*/id,person_img_url from person_pass where db_name='yh' and es_time between '2020-06-17 10:45:35' and '2020-06-24 00:00:00' order by es_time desc

会返回一个scrollid,然后将id放到url中获取后续数据,一直到拿到的数据为空代表读取完了 ip:9200/_search/scroll?scroll=1m&scroll_id=DnF1ZXJ5VGhlbkZldGNoBQAAAAAAEwMtFlMxUng2Q0pwUkNxTV9yZ0plWktiLXcAAAAAABMDLhZTMVJ4NkNKcFJDcU1fcmdKZVpLYi13AAAAAAARXskWMklrckhac0ZSX3lGN0pfSnZjVWJjQQAAAAAAEV7KFjJJa3JIWnNGUl95RjdKX0p2Y1ViY0EAAAAAABJAKxZUeEMtTTU0elF4eXRTTVMteHJlWFRR