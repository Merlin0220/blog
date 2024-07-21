journalctl -xeu kubelet   查看报错日志
kubectl get pods (-o wide) 获取pod信息
kubectl describe pods 较为详细的获取pod信息
kubectl get pods -l run=my-nginx -o wide  使用label过滤了pod（标签run=my-nginx）

kubectl describe svc ci-service--yisu-backend-casemgr

资源别名：
pods：po
deployments：deploy
services：svc
namespace：ns
nodes：no

强制删除Pod：kubectl delete pods <pod_name> --grace-period=0 --force


Label和Selector命令：
临时创建label：kubectl label po <pod名称> app=hello 
修改已经存在的label：kubectl label po <pod名称> app=hello2 --overwrite
查看pod的label：kubectl get po --show-labels

查找存在app=java的pod：**kubectl get po -l app=java (--show-labels)**，加上-A可以显示命名空间
查找version标签值在1.0.0, 1.1.0, 1.2.0之间的pod：**kubectl get po -l 'version in (1.0.0, 1.1.0, 1.2.0)'**
查找version标签不为1.1.0，且type标签为app的pod：**kubectl get po -l version!=1.1.0,type=app**


