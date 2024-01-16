GN可以看作是[[BatchNormalization]]和[[InstanceNormalization]]的结合，其差异如下图所示：
![[assets/Pasted image 20240116200159.png]]
所以可以将LN和IN看作是GN的极端表现。一般而言，我们要求Group的G取值是C的整数倍。

yu