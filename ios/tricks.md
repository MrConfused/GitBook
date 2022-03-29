# Tricks

* 在`.mm`文件中想用`@import`方式，需要在.xcodeproj->target->BuildPhases->对应的文件后面的CompolerFlags加上`-fcxx-modules`
