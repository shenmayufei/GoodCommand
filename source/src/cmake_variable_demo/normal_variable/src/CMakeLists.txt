cmake_minimum_required(VERSION 3.0)
message("进入子目录 src/CMakeLists.txt")
message("设置MY_VAL=777")
set(MY_VAL "777")
message("在子目录, MY_VAL =${MY_VAL}")
message("设置MY_VAL=888 PARENT_SCOPE")
set(MY_VAL "888" PARENT_SCOPE)
message("在子目录, MY_VAL =${MY_VAL}")
message("退出子目录")
