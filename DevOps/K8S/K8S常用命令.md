journalctl -xeu kubelet   查看报错日志
kubectl get pods (-o wide) 获取pod信息
kubectl describe pods 较为详细的获取pod信息
kubectl get pods -l run=my-nginx -o wide  使用label过滤了pod（标签run=my-nginx）

kubectl describe svc ci-service--yisu-backend-casemgr