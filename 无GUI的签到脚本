import pyautogui
from apscheduler.schedulers.blocking import BlockingScheduler
from datetime import datetime
import logging

# 配置日志（可选，方便排查执行失败问题）
logging.basicConfig(
    filename="qiandao_log.txt",  # 日志文件保存路径
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    encoding="utf-8"
)

def qiandao():
    """原有签到逻辑（保持不变）"""
    try:
        pyautogui.FAILSAFE = True
        pyautogui.PAUSE = 0.5
        screen_width, screen_height = pyautogui.size()
        print(f"屏幕分辨率：{screen_width} x {screen_height}")
        logging.info(f"开始签到，屏幕分辨率：{screen_width} x {screen_height}")

        # 移动鼠标到浏览器图标并点击
        pyautogui.click((508, 1055))
        pyautogui.sleep(3)

        # 点击书签
        pyautogui.click(1385, 116)
        pyautogui.sleep(0.5)
        # 点击游戏子标签
        pyautogui.click(1446, 168)
        pyautogui.sleep(2)
        pyautogui.click(1131, 340)

        # 二次点击，防止第一次打不开网页
        pyautogui.click(1385, 116)
        pyautogui.sleep(0.5)
        pyautogui.click(1446, 168)
        pyautogui.sleep(2)
        pyautogui.click(1131, 340)

        print('等待游戏网站页面加载 6秒倒计时开始')
        pyautogui.sleep(6)
        print('网页加载6秒结束，开始点击签到标签')
        pyautogui.click(1870, 713)
        print('等待游戏网站签到标签加载 8秒倒计时开始')
        pyautogui.sleep(8)
        print('签到卡片8秒等待结束，开始点击签到')
        pyautogui.click(1740, 302)

        pyautogui.sleep(2)
        screenshot_path = f"qiandao_screenshot_{datetime.now().strftime('%Y%m%d')}.png"
        pyautogui.screenshot(screenshot_path)
        print(f"签到完成！截图已保存为：{screenshot_path}")
        logging.info(f"签到成功，截图路径：{screenshot_path}")

    except Exception as e:
        # 捕获异常并记录到日志（方便排查问题）
        error_msg = f"签到失败：{str(e)}"
        print(error_msg)
        logging.error(error_msg)

if __name__ == '__main__':
    # 创建定时调度器
    scheduler = BlockingScheduler()

    # 添加任务：每天早上8:30执行签到
    scheduler.add_job(
        func=qiandao,
        trigger="cron",  # Cron触发器（支持固定时间）
        hour=8,          # 小时：8点
        minute=30,       # 分钟：30分
        id="daily_qiandao",
        replace_existing=True,
        misfire_grace_time=300  # 允许延迟5分钟执行（防止电脑略卡导致错过时间）
    )

    print(f"定时签到脚本已启动！每天8:30自动执行\n日志文件：qiandao_log.txt")
    logging.info("定时签到脚本启动，每天8:30执行")

    try:
        scheduler.start()  # 启动调度器（阻塞运行，不要关闭终端）
    except KeyboardInterrupt:
        # 按 Ctrl+C 停止脚本
        scheduler.shutdown()
        print("脚本已手动停止")
        logging.info("脚本手动停止")
