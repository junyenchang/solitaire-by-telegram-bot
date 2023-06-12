# Projects：telegram上的踩地雷小遊戲
透過telegram bot，使用python實作一個可以在telegram上操作互動的接龍小遊戲

## 說明
* 輸入`/start`指令產生初始的牌，會出現以下資訊：
```
♣(0): 
♦(1): 
❤️(2): 
♠(3): 
🂠(4): 
row0(5):♣1
row1(6):♦1 *
row2(7):♠1 * *
row3(8):❤️2 * * *
row4(9):❤️3 * * * *
row5(10):♠4 * * * * *
row6(11):♦6 * * * * * *
```
1. 第 0~3 列為示意圖中右上角，最後的依序放牌的答案區。只能放該花色的牌且由小排到大。
2. 第 4 列為抽牌區，此處覆蓋第 3. 分完後剩下的牌。每次只能翻開最上面那張牌。
3. 第 5~11 列為示意圖中下面暫時放的牌區，初始於最左列產生一張已翻開的牌以及每往右一列就多加一張覆蓋的牌。每列只能由小排到大，但不需有花色限制。

* `/move`
    輸入參數：
    第一個輸入為要移動第幾列的牌。
    第二個輸入為要移動的牌組的第幾張牌。
    第三個輸入為要移動至哪一個列。

    移動規則：
        1. 只能移動已經翻開的牌。
        2. 從已經翻開且連續排放的牌中指定位置，移動時為指定位置及前面位置的牌一起移動。
        3. 移動至指定列（第三個輸入，答案區，只能由小到大排放）。
        4. 若欲移動之牌組數字無法與指定列連續，則無法進行移動，並顯示移動失敗。
        5. 若移動完所有的牌至答案區，則顯示遊戲成功訊息。

* `翻牌按鈕`
需使用 `InlineKeyboardButton` 建立一個翻牌按鈕，功能為一次翻出一張牌。

* `重玩按鈕`
需使用 `InlineKeyboardButton` 建立一個重完按鈕，功能為恢復初始的狀態(/start)。


## 輸出說明
* 每張牌顯示為 ♣、♦、❤️、♠ 這四個 icon 配上 1~13 的數字（為計算及顯示方便不用轉換成(A、J、Q、K)。
* 未翻開的牌以 `*` 表示。


## 範例
* 輸入：
```
/start
```
產生初始的牌面
* 輸出：
![](https://hackmd.io/_uploads/HJfjoBVDn.png)
* 點擊翻牌按鈕
![](https://hackmd.io/_uploads/HyUGpHVvn.png)

* 輸入：
```
/move 4 0 10
```
將畫面上第 4 列的第 0 張牌（最上面那張）移動至畫面上第 10 列(row5)。

![](https://hackmd.io/_uploads/S1U6sB4v3.png)


## 實作
### import
```python=
from telegram.ext import Updater, CommandHandler, CallbackQueryHandler
from telegram import InlineKeyboardMarkup, InlineKeyboardButton
from random import sample
```
### 連接telegram bot
```python=
with open('token.txt') as FILE:
    token = FILE.read().strip('\n')

updater = Updater(token=token, use_context=True)
dispatcher = updater.dispatcher
```
 
### 準備牌庫
```python=
row_4 = []  # 抽牌用的 第四行
card = [f"♣{i}"for i in range(1, 14)] + [f"♦{i}"for i in range(1, 14)] + [f"❤{i}"for i in range(1, 14)] + [f"♠{i}"for i in range(1, 14)]  # 一副撲克牌


def pop():  # 從card抽一張牌
    c = sample(card, 1)
    card.remove(c[0])
    return c


rows_head = ["♣(0):", "♦(1):", "❤(2):", "♠(3):", "🂠(4):"]+[f"row{i}({i+5}):" for i in range(7)]
rows_tail = ["", "", "", "", "", ' '+''.join(pop()[0]), ' '+''.join(pop()[0])+" *", ' '+''.join(pop()[0])+" * *", ' '+''.join(pop()[0])+" * * *",
             ' '+''.join(pop()[0])+" * * * *", ' '+''.join(pop()[0])+" * * * * *", ' '+''.join(pop()[0])+" * * * * * *"]
```
### Define function
#### 初始化盤面
```python=
def init(update, context):  # start 指令執行
    context.bot.send_message(
        chat_id=update.effective_chat.id,
        text='\n'.join([head+tail for head, tail in zip(rows_head, rows_tail)]),
        reply_markup=InlineKeyboardMarkup([[
            InlineKeyboardButton(
                '翻牌', callback_data='A'),
            InlineKeyboardButton(
                '重玩', callback_data='B')
        ]])
    )
```
#### 檢查輸入的指令格式是否合法
```python=
def check():    # 判斷輸入的移動指令是否合法
    global error, A, B, row, to  # 判斷輸入的move是否合法
    error = 0
    x = A[-1][1:]   # 號碼
    f = A[-1][0]    # 花色
    y = B
    w = int(x)  # 牌的號碼
    z = int(y)  # 目的地是否有東西

    if 0 <= row <= 3:   # 如果是從0~3堆移東西
        if 5 <= to <= 11:   # 移往5~11堆
            if w == 13 and z == 0:  # 號碼是13 但目的地沒東西，所以可以
                error = error
            elif w == 13 and z != 0:    # 因為要由小排到大，如果目的地有東西就不行
                error = 1
            elif z == 0:    # 目的地是空的就沒問題
                error = error
            elif z != w + 1:    # 要照順序排才可以
                error = 2
        else:   # 因為只能放特定花色，所以也是移到0~3堆不合法
            error = 3
    elif 4 <= row <= 11:   # 如果是從4~11堆移東西
        if 5 <= to <= 11:   # 移往5~11堆
            if w == 13 and z == 0:
                error = error
            elif w == 13 and z != 0:
                error = 4
            elif z != w+1:
                error = 5
        elif to == 4:  # 移往第四堆不合法，四只能移出
            error = 6
        elif 0 <= to <= 3:  # 移往第0~3堆
            if z == 0 and w != 1:   # 一開始只能放 1號牌
                error = 7
            elif w != z + 1:    # 照順序排
                error = 8
            elif f != rows_head[to][0]:  # 跟目的地花色要一樣
                error = 9
    print(f'error(check): {error}')
```
#### 處理收到指令後要做的動作
```python=
def move(update, context):
    global error
    step = update.message.text.split(' ')
    mistake = 0  # 偵錯用的
    if len(step) != 4:  # 簡單的防呆 偵測輸入格式是否正確
        context.bot.send_message(
            chat_id=update.effective_chat.id,
            text="Invalid input"
        )
        return
    else:
        global row, to
        row = int(step[1])  # 要移動哪一行
        num = int(step[2])  # 的第幾張牌
        to = int(step[3])  # 要移往第幾行
        global A, B
        A = rows_tail[row].split()[0:num+1]  # 把要移動的牌後面都切出來
        C = rows_tail[to].split()   # 把目的地的東西 切開備用
        if len(A) == 0:  # A 沒東西不合法
            mistake = 1
        elif "*" in A:      # 不能移動蓋著的牌
            mistake = 2
        elif row < 0 or row > 11:
            mistake = 3
        elif to < 0 or to > 11:
            mistake = 4
        elif row == to:
            mistake = 5
        elif to == 4:
            mistake = 6
        elif 0 <= to <= 3 and num > 0:
            mistake = 7
        elif row == 4 and num != 0:     # 從抽牌堆移牌必定只能移第一張
            mistake = 8
        print(f'mistake: {mistake}')
        if mistake != 0:  # 簡單的防呆 偵測輸入格式是否正確
            context.bot.send_message(
                chat_id=update.effective_chat.id,
                text="Invalid input"
            )
            return
        else:
            if len(C) == 0:
                B = "0"
            else:
                B = rows_tail[to].split()[0][1:]
            check()
        if row == 4 and error == 0:  # 從抽牌堆移動的話
            card.remove(rows_tail[4])
        if error == 0:
            rows_tail[row] = ' '+' '.join(rows_tail[row].split()[num + 1:])  # 把牌合併在一起
            if rows_tail[row][:2] == ' *':  # 第一張牌是蓋著的話，就抽一張牌
                New = pop()[0]
                rows_tail[row] = ' '+''.join(New + rows_tail[row][2:])      # 把抽出來的牌接上去
            else:
                rows_tail[row] = rows_tail[row]
            rows_tail[to] = ' '+' '.join(A + rows_tail[to].split())  # 把移過去的牌合併
            context.bot.send_message(
                chat_id=update.effective_chat.id,
                text='\n'.join([head+tail for head, tail in zip(rows_head, rows_tail)]),
                reply_markup=InlineKeyboardMarkup([[
                    InlineKeyboardButton(
                        '翻牌', callback_data='A'),
                    InlineKeyboardButton(
                        '重玩', callback_data='B')
                ]])
            )
        elif error > 0 and mistake == 0:
            context.bot.send_message(
                chat_id=update.effective_chat.id,
                text="Invalid input"
            )
        if len(card) == 0:  # 把所有排翻出即算勝利
            context.bot.send_message(
                chat_id=update.effective_chat.id,
                text="Finish!!! You win~~~"
            )
```

#### 按下按鍵後對應發生的事情(抽牌、重玩)
```python=
def button(update, context):    # 按下兩個按鍵會對應到的function
    global rows_tail, card
    r_5 = rows_tail[5].split().count("*")
    r_6 = rows_tail[6].split().count("*")
    r_7 = rows_tail[7].split().count("*")
    r_8 = rows_tail[8].split().count("*")
    r_9 = rows_tail[9].split().count("*")
    r_10 = rows_tail[10].split().count("*")
    r_11 = rows_tail[11].split().count("*")
    total = r_5 + r_6 + r_7 + r_8 + r_9 + r_10 + r_11   # 計算盤面還有幾張牌蓋著
    if update.callback_query.data == 'A':   # 按 抽牌
        if len(card) <= total:  # 牌庫牌的數量小於場上還覆蓋著的牌，就不能再翻牌
            context.bot.send_message(
                chat_id=update.effective_chat.id,
                text="No cards in the pool !"
            )
        else:
            rows_tail[4] = sample(card, 1)[0]
            context.bot.send_message(
                chat_id=update.effective_chat.id,
                text='\n'.join([head + tail for head, tail in zip(rows_head, rows_tail)]),
                reply_markup=InlineKeyboardMarkup([[
                    InlineKeyboardButton(
                        '翻牌', callback_data='A'),
                    InlineKeyboardButton(
                        '重玩', callback_data='B')
                ]])
            )
    elif update.callback_query.data == 'B':     # 重新產生盤面，重製牌堆
        card = [f"♣{i}" for i in range(1, 14)] + [f"♦{i}" for i in range(1, 14)] + [f"❤{i}" for i in range(1, 14)] + [
            f"♠{i}" for i in range(1, 14)]
        rows_tail = ["", "", "", "", "", ' ' + ''.join(pop()[0]), ' ' + ''.join(pop()[0]) + " *",
                     ' ' + ''.join(pop()[0]) + " * *", ' ' + ''.join(pop()[0]) + " * * *",
                     ' ' + ''.join(pop()[0]) + " * * * *", ' ' + ''.join(pop()[0]) + " * * * * *",
                     ' ' + ''.join(pop()[0]) + " * * * * * *"]
        context.bot.send_message(
            chat_id=update.effective_chat.id,
            text='\n'.join([head + tail for head, tail in zip(rows_head, rows_tail)]),
            reply_markup=InlineKeyboardMarkup([[
                InlineKeyboardButton(
                    '翻牌', callback_data='A'),
                InlineKeyboardButton(
                    '重玩', callback_data='B')
            ]])
        )
```
#### 開始在telegram上玩遊戲
```python=
start_handler = CommandHandler('start', init)
dispatcher.add_handler(start_handler)
dispatcher.add_handler(CallbackQueryHandler(button))

move_handler = CommandHandler('move', move)
dispatcher.add_handler(move_handler)

updater.start_polling()
```
## Demo
* 程式碼講解 :  https://youtu.be/DMIMbd53C9w
* 程式執行畫面 : https://youtu.be/UQlK5SUbMiM


---

###### tags: `telegram bot`、`Python`、`Solitaire`