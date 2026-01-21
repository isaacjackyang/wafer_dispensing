# wafer_dispensing
its a html GUI
generator dispensing dots for machine movement


1. HTML 結構原理

    Layer 概念 (三明治架構)：

        #world：這就像是我們的「桌子」。我們在上面放東西。

        #layer-video (底層)：就像一張墊在最下面的照片 (Webcam 影像)。

        #layer-grid (中層)：像是一張透明的方格紙 (網格)，幫我們對齊位置。

        #layer-draw (上層)：像是最上面那張透明描圖紙，我們用筆在上面畫線 (SVG 繪圖)。

        為什麼要分層？ 因為 SVG 畫多了會變慢，把網格和影片分開處理，效能會更好，而且不會互相干擾（比如想把網格關掉，就直接隱藏那一層就好）。

    Viewport (視窗)：

        #canvas-window：這是我們眼睛看到的「視窗框框」。

        #world：這是在框框裡面的「世界」。

        當我們「放大」時，其實是把 #world 變大；當我們「平移」時，是把 #world 往旁邊推。視窗框框本身是不動的。

2. CSS 樣式原理

    position: absolute：絕對定位。這讓我們的圖層可以疊在一起 (top:0, left:0)，如果沒有這個，它們會排成一列。

    pointer-events: none：重要！ 這是「穿透」屬性。

        我們把 Video 和 Grid 層設為 none，這樣滑鼠點擊時，瀏覽器會忽略它們，直接點到最下面的事件監聽器 (或最上層的繪圖層)，才不會因為影片擋住而畫不出來。

    vector-effect: non-scaling-stroke：神器！

        一般的線條放大 10 倍，粗細也會變 10 倍，變成一條巨蟒。

        加了這個屬性後，SVG 會自動調整，讓線條不管怎麼放大縮小，看起來都維持原本的 1px 細度。

3. JavaScript 核心邏輯
A. 視圖變換 (Pan & Zoom)

這段代碼是模仿 CAD 軟體的操作手感：

    平移 (Pan)：

        記錄滑鼠按下去的位置 (startX)。

        滑鼠移動時，計算移動了多少距離 (dx)。

        把 #world 的位置 (view.x) 加上這個距離。

    縮放 (Zoom)：

        這是一個經典的數學公式：「以滑鼠為中心縮放」。

        view.x = mx - (mx - view.x) * (newScale / oldScale)

        這句話的意思是：縮放前後，滑鼠指著的那個點 (在世界地圖上的位置) 必須保持在螢幕的同一個位置不動。這樣縮放起來才自然，不會一放大目標就跑掉了。

B. 座標轉換 (Coordinate Transform)

    螢幕座標 (Screen)：滑鼠在瀏覽器上的位置 (px)。

    世界座標 (World/mm)：我們真實想要紀錄的尺寸 (mm)。

    轉換公式：

        World_X = (Screen_X - Pan_X) / Zoom / Scale

        先扣掉平移量 (Pan)，再除以縮放倍率 (Zoom)，最後除以校正比例 (Scale, 1mm=多少px)。

        這樣算出來的 (x, y) 才是真實的物理座標。

C. 手繪邏輯 (Manual Draw)

    狀態機 (State Machine)：

        manPts 陣列存著所有的點。

        drag 物件存著「現在有沒有在拖曳」。

    流程：

        mousedown：檢查滑鼠是不是點在某個舊點附近 (dist < 5)？

            是 -> 進入「拖曳模式」。

            否 -> 進入「新增模式」，加一個新點進 manPts。

        mousemove：

            如果是拖曳模式 -> 更新那個點的座標。

        renderManual()：

            這是一個「畫家」。每次資料有變動 (新增或移動點)，就叫畫家把整個畫面擦掉重畫一次。雖然聽起來笨，但在電腦上這比「只修改一條線」更不容易出錯，也夠快。

D. 統計計算 (Stats)

    資料流：

        自動模式 -> 算出 pathData。

        手繪模式 -> 算出 pathData。

        統一出口 -> calcStats(pathData)。

    這就是為什麼之前的版本會失效，因為資料流分岔了。現在不管哪種模式，最後都匯總成一個標準的 pathData 陣列給計算機算，保證準確。

(示意圖：最底層是 Video，中間是 Grid，上層是 Draw，三者被包在一個 World Group 裡一起被縮放)

(示意圖：滑鼠位置不變，世界座標系相對滑鼠進行擴張或收縮)
