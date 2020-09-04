# MLflow手册

MLflow是一个开源的管理端到端机器学习生命周期的平台。包含4个主要的功能：

* MLflow Tracking：追溯实验用来记录、比较参数和结果。
* MLflow Projects：将ML代码打包成一个可重用、可复现的形式，可以共享给其他数据科学家或者部署到生产。
* MLflow Models：管理模型或者将模型部署到各种模型服务或者推理平台。
* Mlflow Model Registry：提供一个模型存储中心来管理MLflow model的全生命周期，包括model versioning, stage transitions, and annotations

MLflow是框架无关的（ library-agnostic）。因为所有功能可以通过REST API和命令行访问，所以可以结合任意机器学习库、任意编程语言使用。为了方便使用，也提供了python、R和java API。



