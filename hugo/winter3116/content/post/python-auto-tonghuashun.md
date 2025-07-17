---
title: "Python Auto Tonghuashun"
date: 2025-07-18T00:10:21+08:00
draft: true
---

### 代码
```python
from pywinauto import Application
from pywinauto import keyboard
import time
import logging
import pyautogui
import chinese_calendar as calendar
import datetime


# 关闭安全模式
pyautogui.FAILSAFE = False

# 配置日志（可选）
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s: %(message)s',
    handlers=[logging.FileHandler("ths_automation.log"), logging.StreamHandler()],
    encoding="utf-8"
)

def main():
    if not marketstate():
        logging.info("today holiday")
        return
    if tonghuashun():
        logging.info("success")
    else:
        logging.info("failed")

def marketstate():
    week = int(time.strftime("%W", time.localtime()))
    is_holiday = calendar.is_holiday(datetime.date.today())
    if is_holiday:
        return False
    if week == 6 or week == 0:
        return False
    return True

def tonghuashun():
    try:
        # 1. 启动或连接同花顺应用
        try:
            app = Application(backend="uia").connect(title_re="同花顺.*", class_name_re="Afx.*", framework_id="Win32", found_index=0, timeout=10)
            logging.info("tonghuashun connect...")
        except Exception as e:
            logging.warning(str(e))
            app = Application(backend="uia").start(r"D:\同花顺\hexin.exe", timeout=10)  # 替换为实际安装路径
            time.sleep(10)
            logging.info("tonghuashhun start...")

        # 2. 获取主窗口
        time.sleep(1)
        main_win = app.windows(title_re="同花顺.*")[0]
        logging.info("into main window...")

        main_win.maximize()
        time.sleep(0.5)
        keyboard.send_keys("%e")
        time.sleep(0.5)
        main_win.restore()
        logging.info("into my choose window...")

        my_block_win = main_win.children(class_name="AfxFrameOrView140s", control_type="Pane")[0].descendants(class_name="block_list_page", control_type="Pane")[0]
        # 复杂的页面还是无法定位

        ###############################################################
        rect = my_block_win.rectangle()

        hight = rect.bottom - rect.top
        width = rect.right - rect.left

        logging.info(f"in my block window..., rect:{str(rect)}")

        # 3 处理“前日涨停”和“昨日涨停”
        pyautogui.moveTo(rect.right-3, rect.top+20)
        pyautogui.doubleClick()

        qrzt = search_center("./png/qrzt.png", region=(rect.left, rect.top, width, hight))
        if qrzt is None:
            qrzt = search_center("./png/qrzt_cl.png")
        if qrzt:
            del_block(qrzt, rect, main_win)
            logging.info("qrzt success...")

        zrzt = search_center("./png/zrzt.png", region=(rect.left, rect.top, width, hight))
        if zrzt is None:
            zrzt = search_center("./png/zrzt_cl.png")
        if zrzt:
            add_block(zrzt, rect, main_win, "前日涨停")
            del_block(zrzt, rect, main_win)
            logging.info("zrzt success...")

        # 4 处理动态组
        pyautogui.moveTo(rect.right - 3, rect.bottom - 40)
        pyautogui.doubleClick()

        ztjw = search_center("./png/ztjw.png", region=(rect.left, rect.top, width, hight))
        if ztjw is None:
            ztjw = search_center("./png/ztjw_cl.png")
        if ztjw:
            add_block(ztjw, rect, main_win, "昨日涨停")
            logging.info("ztjwt success...")

        jreb = search_center("./png/jreb.png", region=(rect.left, rect.top, width, hight))
        if jreb is None:
            jreb = search_center("./png/jreb_cl.png")
        if jreb:
            add_block(jreb, rect, main_win, "二三板")
            logging.info("jreb success...")

        return True
    except Exception as e:
        logging.error(f"操作失败: {str(e)}")
        # 建议此处添加错误截图功能 [1](@ref)
        return False

def search_center(image_path, region=None):
    button_center = None
    try:
        if region is None:
            button_location = pyautogui.locateOnScreen(image_path, minSearchTime=120)
        else:
            button_location = pyautogui.locateOnScreen(image_path, minSearchTime=60, region=region)
        logging.info(button_location)
        if button_location is not None:
            button_center = pyautogui.center(button_location)
        else:
            logging.error("button not found")
    except Exception as e:
        logging.error(str(e))
    return button_center

def del_block(point, rect, main_win):
    pyautogui.moveTo(point)
    pyautogui.click()
    time.sleep(1)
    pyautogui.doubleClick()

    pyautogui.moveTo(rect.right + 60, rect.top + 60)
    pyautogui.click()
    pyautogui.rightClick()

    plcz = search_center("./png/dmplcz.png")
    if plcz:
        pyautogui.moveTo(plcz)
        pyautogui.click(plcz)

        plcz_win = main_win.children(class_name="#32770", control_type="Window")[0]
        choose_button = plcz_win.children(title="全选中", control_type="Button")[0]
        del_button = plcz_win.children(title="从板块删除", control_type="Button")[0]

        choose_button.click()
        del_button.click()

def add_block(point, rect, main_win, block_name):
    pyautogui.moveTo(point)
    pyautogui.click()
    time.sleep(0.5)
    pyautogui.doubleClick(point)
    time.sleep(3) # 动态组更新

    pyautogui.moveTo(rect.right + 60, rect.top + 60)
    pyautogui.click()
    pyautogui.rightClick()

    plcz = search_center("./png/dmplcz.png")
    if plcz:
        pyautogui.moveTo(plcz)
        pyautogui.click(plcz)

        plcz_win = main_win.children(class_name="#32770", control_type="Window")[0]
        choose_button = plcz_win.children(title="全选中", control_type="Button")[0]
        add_button = plcz_win.children(title="加入到板块", control_type="Button")[0]

        choose_button.click()
        add_button.click()

        time.sleep(1)
        choose_block_win = plcz_win.children(title="选择板块", control_type="Window")[0]

        yes_button = choose_block_win.children(title="确定", control_type="Button")[0]
        gz_block = choose_block_win.descendants(title=block_name, control_type="TreeItem")[0]

        gz_block.select()
        rect_block = gz_block.rectangle()
        pyautogui.moveTo((rect_block.left + rect_block.right) // 2, (rect_block.top + rect_block.bottom) // 2)
        pyautogui.click()
        yes_button.click()

if __name__ == '__main__':
    main()
```