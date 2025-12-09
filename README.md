<img width="643" height="305" alt="image" src="https://github.com/user-attachments/assets/7b4e65b0-59d3-4bd3-a33b-6c9f0e8ec9e2" /># xbgame_Sign_in
小白游戏网自动签到脚本

功能说明：
  通过python脚本，操作鼠标，进行坐标的点击操作。
  在图形化脚本中使用定时器，可自定义自动签到的时间。

效果：
  当到达设定时间时，自动点击相关坐标，完成签到并截图保存。

打包：
  pyinstaller -F -w --name "自动签到助手" --add-data "qiandao_config.ini;." --add-data "screenshots;screenshots" --icon=ico.ico .\qiandao_GUI.py


![Uploading 主页面.PNG…]()
