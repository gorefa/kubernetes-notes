在K8S的持久化存储中主要有以下几种分类：

- volume: 就是直接挂载在pod上的组件，k8s中所有的其他存储组件都是通过volume来跟pod直接联系的。volume有个type属性，type决定了挂载的存储是什么，常见的比如：emptyDir，hostPath，nfs，rbd，以及下文要说的persistentVolumeClaim等。跟docker里面的volume概念不同的是，docker里的volume的生命周期是跟docker紧紧绑在一起的。这里根据type的不同，生命周期也不同，比如emptyDir类型的就是跟docker一样，pod挂掉，对应的volume也就消失了，而其他类型的都是永久存储。详细介绍可以参考Volumes
- Persistent Volumes：顾名思义，这个组件就是用来支持永久存储的，Persistent Volumes组件会抽象后端存储的提供者（也就是上文中volume中的type）和消费者（即具体哪个pod使用）。该组件提供了PersistentVolume和PersistentVolumeClaim两个概念来抽象上述两者。一个PersistentVolume（简称PV）就是后端存储提供的一块存储空间，具体到ceph rbd中就是一个image，一个PersistentVolumeClaim（简称PVC）可以看做是用户对PV的请求，PVC会跟某个PV绑定，然后某个具体pod会在volume 中挂载PVC,就挂载了对应的PV。
- Dynamic Volume Provisioning: 动态volume发现，比如上面的Persistent Volumes,我们必须先要创建一个存储块，比如一个ceph中的image，然后将该image绑定PV，才能使用。这种静态的绑定模式太僵硬，每次申请存储都要向存储提供者索要一份存储快。Dynamic Volume Provisioning就是解决这个问题的。它引入了StorageClass这个概念，StorageClass抽象了存储提供者，只需在PVC中指定StorageClass，然后说明要多大的存储就可以了，存储提供者会根据需求动态创建所需存储快。甚至于，我们可以指定一个默认StorageClass，这样，只需创建PVC就可以了。





[K8S与Ceph RBD集成](https://blog.51cto.com/tryingstuff/2386821)

[使用Ceph RBD为Kubernetes集群提供存储卷](https://tonybai.com/2016/11/07/integrate-kubernetes-with-ceph-rbd/)


