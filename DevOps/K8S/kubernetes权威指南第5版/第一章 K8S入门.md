### Pod如何与Service进行关联
Kubernetes 首先给每个 Pod 都贴上一个标签(Label)，比如给运行 MySOL 的 Pod 贴上 name=mysql 标签，给运行PHP 的Pod贴上 name=php标签。
然后给相应的 Service **定义标签选择器(LabelSelector)**，例如，MySQL Service 的标签选择器的选择条件为name=mysql，意为该 Service 要作用于所有包含 name=mysql 标签的 Pod。这样一来，就巧妙解决了 Service 与 Pod 的关联问题。