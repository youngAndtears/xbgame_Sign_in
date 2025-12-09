

功能说明：
  通过python脚本，操作鼠标，进行坐标的点击操作。
  在图形化脚本中使用定时器，可自定义自动签到的时间。

效果：
  当到达设定时间时，自动点击相关坐标，完成签到并截图保存。

打包：
  pyinstaller -F -w --name "自动签到助手" --add-data "qiandao_config.ini;." --add-data "screenshots;screenshots" --icon=ico.ico .\qiandao_GUI.py


<img width="1002" height="790" alt="image" src="https://github.com/user-attachments/assets/daf553c0-4327-495f-9600-83d20a8b158f" />


