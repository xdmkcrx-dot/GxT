//@version=6
indicator("ALCHEMIST ANALYZER V3.2", "ALCHEMIST V3.2", overlay = true,
     max_lines_count = 300, max_labels_count = 300, max_boxes_count = 200)

//====================================================================
// ALCHEMIST ANALYZER V3.2  (fokus Time & Price, GxT Sequence, tanpa SMT)
// Alat analisa saja. Tidak ada buy/sell. Alert hanya untuk inversi IFVG.
//====================================================================

// ------------------------------ INPUT ------------------------------
string G1  = "01 · HTF LEVELS"
string G2  = "02 · STRUCTURE"
string G3  = "03 · LIQUIDITY"
string G4  = "04 · DEALING RANGE"
string G5  = "05 · KILL ZONE (GxT)"
string G6  = "06 · TIME · MACRO & SILVER BULLET (NY)"
string G7  = "07 · TAMPILAN & SEQUENCE"
string G8  = "08 · OPENING PRICES"
string G9  = "09 · PO3"
string G10 = "10 · FVG / IFVG"

string htf1   = input.timeframe("60",  "HTF 1", group = G1)
string htf2   = input.timeframe("240", "HTF 2", group = G1)
bool showPDH  = input.bool(true,  "PDH / PDL", group = G1)
bool showPWH  = input.bool(true,  "PWH / PWL", group = G1)
bool showOCL2 = input.bool(true,  "OCL HTF 2 (Open/Close candle sebelumnya)", group = G1)
bool showOCL1 = input.bool(false, "OCL HTF 1", group = G1)
int  lineExt  = input.int(8, "Panjang garis ke kanan (bar)", minval = 2, maxval = 40, group = G1)

int  pivotLen = input.int(5, "Pivot Length (struktur)", minval = 2, maxval = 20, group = G2)
int  htfPivot = input.int(3, "Pivot Length (bias HTF)", minval = 1, maxval = 10, group = G2)
bool showBOS  = input.bool(true,  "BOS / MSS", group = G2)
bool showSwing = input.bool(false, "Label HH / HL / LH / LL", group = G2)
int  maxMarks = input.int(20, "Maks. tanda di chart (BOS/MSS/EQ/Sweep)", minval = 4, maxval = 60, group = G2)

bool  showEQ     = input.bool(true, "EQH / EQL", group = G3)
float eqTol      = input.float(0.10, "Toleransi EQ × ATR", minval = 0.01, maxval = 1.0, step = 0.01, group = G3)
bool  showSweeps = input.bool(true, "Liquidity Sweep (swing, PDH, PDL)", group = G3)
int   sessRevWin = input.int(48, "Reversal: jendela sapuan high/low sesi (bar)", minval = 6, maxval = 300, group = G3)

bool showPD  = input.bool(true, "Premium / Discount + EQ 50%", group = G4)
bool showOTE = input.bool(true, "OTE 0.62–0.79 (ideal 0.705)", group = G4)

bool   showSess = input.bool(true, "Tampilkan Kill Zone", group = G5)
string sessTZ   = input.string("Asia/Jakarta", "Timezone", options = ["Asia/Jakarta", "Etc/UTC", "Europe/London", "America/New_York"], group = G5)
string asiaS    = input.session("0700-1200", "Asia KZ", group = G5)
string londonS  = input.session("1400-1900", "London KZ", group = G5)
string nyS      = input.session("1900-0000", "New York KZ", group = G5)
int    maxSess  = input.int(4, "Maks. kotak per sesi", minval = 1, maxval = 10, group = G5)

bool   showICTMacro = input.bool(true, "ICT Macros (07:50–08:10 dst.)", group = G6)
bool   showSBMacro  = input.bool(true, "Macro SB (xx:20–xx:40 & xx:50–xx:10, di dalam jendela SB)", group = G6)
bool   showSBBox    = input.bool(true, "Kotak jendela Silver Bullet", group = G6)
bool   showKeyTimes = input.bool(true, "Garis open xx:15 & xx:45 (di dalam SB)", group = G6)
string sbLdnS = input.session("0300-0400", "SB London (waktu NY)", group = G6)
string sbPreS = input.session("0700-0830", "SB NY Premarket (waktu NY)", group = G6)
string sbAmS  = input.session("1000-1100", "SB NY AM (waktu NY)", group = G6)
string sbPmS  = input.session("1400-1500", "SB NY PM (waktu NY)", group = G6)
string sbFinS = input.session("1500-1600", "SB PM Finish (waktu NY)", group = G6)
color  colMacro = input.color(#9c27b0, "Warna macro / SB", group = G6)

bool   showDash = input.bool(true, "Dashboard", group = G7)
string dashPos  = input.string("Top Right", "Posisi dashboard", options = ["Top Right", "Top Left", "Bottom Right", "Bottom Left"], group = G7)
color  colBull  = input.color(#089981, "Warna bullish", group = G7)
color  colBear  = input.color(#f23645, "Warna bearish", group = G7)
color  colLvl   = input.color(#787b86, "Warna level", group = G7)
float  seqExp   = input.float(1.0, "Candle D1 ekspansi: body × ATR D1", minval = 0.3, maxval = 3.0, step = 0.1, group = G7)

bool   showDO  = input.bool(true,  "Daily Open", group = G8)
bool   showWO  = input.bool(true,  "Weekly Open", group = G8)
bool   showMO  = input.bool(false, "Monthly Open", group = G8)
bool   showH2O = input.bool(true,  "HTF 2 Open (candle berjalan)", group = G8)
bool   showTimeOpens = input.bool(true, "Waktu open custom (hanya TF ≤ 1H)", group = G8)
string openTZ  = input.string("America/New_York", "Timezone waktu open", options = ["America/New_York", "Asia/Jakarta", "Etc/UTC", "Europe/London"], group = G8)
string openList = input.text_area("0000,00:00\n0030,00:30\n1800,18:00\n2200,22:00", "Daftar waktu: HHmm,Label (satu per baris, maks. 8)", group = G8)
color  colOpen = input.color(#f5a623, "Warna garis open", group = G8)

bool   showPO3    = input.bool(true, "Tampilkan PO3 Candles", group = G9)
bool   po3Auto    = input.bool(true, "Timeframe PO3 otomatis", group = G9)
string po3Manual  = input.string("60", "Timeframe PO3 manual", options = ["15", "30", "60", "240", "D", "W", "M"], group = G9)
int    po3Offset  = input.int(18, "Jarak candle PO3 dari bar terakhir", minval = 0, maxval = 100, group = G9)
float  po3Thr     = input.float(0.15, "Ambang Judas (× ATR HTF)", minval = 0.01, maxval = 2.0, step = 0.01, group = G9)
bool   showMidWick = input.bool(true,  "0.5 wick candle sebelumnya", group = G9)
bool   projMidWick = input.bool(true,  "Proyeksikan 0.5 wick ke chart", group = G9)
bool   showCE      = input.bool(false, "CE 50% candle sebelumnya", group = G9)
bool   showCount   = input.bool(true,  "Garis + hitung mundur close candle HTF", group = G9)

bool  showFVG  = input.bool(true, "FVG aktif", group = G10)
bool  showIFVG = input.bool(true, "IFVG (FVG yang di-inversi)", group = G10)
float fvgMin   = input.float(0.15, "Ukuran minimum gap × ATR", minval = 0.01, maxval = 2.0, step = 0.01, group = G10)
float dispMin  = input.float(0.8, "Displacement: body candle penembus × ATR", minval = 0.1, maxval = 3.0, step = 0.1, group = G10)
int   maxFVG   = input.int(6, "Maks. zona FVG/IFVG", minval = 2, maxval = 20, group = G10)
int   sweepWin = input.int(30, "Jendela sweep sebelum inversi (bar)", minval = 5, maxval = 100, group = G10)

// ----------------------------- FUNGSI ------------------------------
f_trend(int len) =>
    float ph = ta.pivothigh(high, len, len)
    float pl = ta.pivotlow(low, len, len)
    var float hA = na
    var float hB = na
    var float lA = na
    var float lB = na
    if not na(ph)
        hB := hA
        hA := ph
    if not na(pl)
        lB := lA
        lA := pl
    int t = 0
    if not na(hB) and not na(lB)
        if hA > hB and lA > lB
            t := 1
        else if hA < hB and lA < lB
            t := -1
    t

f_txt(int t) => t == 1 ? "BULLISH" : t == -1 ? "BEARISH" : "MIXED"

f_tf(string tf) =>
    int s = timeframe.in_seconds(tf)
    s >= 2592000 ? "M" : s >= 604800 ? "W" : s >= 86400 ? "D" : s >= 3600 ? str.tostring(s / 3600, "#") + "H" : str.tostring(s / 60, "#") + "m"

f_pos(string s) =>
    switch s
        "Top Left"     => position.top_left
        "Bottom Right" => position.bottom_right
        "Bottom Left"  => position.bottom_left
        => position.top_right

f_in(string s, string tz) => not na(time(timeframe.period, s, tz))

f_mark(int x1, float y1, int x2, float y2, string txt, color c, string sty,
     int lx, float ly, string lst, array<line> la, array<label> lb, int maxN) =>
    line l = line.new(x1, y1, x2, y2, color = c, style = sty, width = 1)
    label b = label.new(lx, ly, txt, style = lst, color = color.new(chart.bg_color, 100), textcolor = c, size = size.tiny)
    array.push(la, l)
    array.push(lb, b)
    if array.size(la) > maxN
        line.delete(array.shift(la))
        label.delete(array.shift(lb))
    true

f_tag(int x, float y, string txt, color c, string lst, array<label> arr, int maxN) =>
    label b = label.new(x, y, txt, style = lst, color = color.new(chart.bg_color, 100), textcolor = c, size = size.tiny)
    array.push(arr, b)
    if array.size(arr) > maxN
        label.delete(array.shift(arr))
    true

f_lbl(int x, float y, string txt, color c, string lst, array<label> arr) =>
    array.push(arr, label.new(x, y, txt, style = lst, color = color.new(chart.bg_color, 100), textcolor = c, size = size.tiny))
    true

f_level(float price, int startIdx, string txt, color c, string sty,
     array<line> la, array<label> lb, int ext) =>
    if not na(price)
        int x1 = math.max(startIdx, bar_index - 4500)
        line l = line.new(x1, price, bar_index + ext, price, color = c, style = sty, width = 1)
        label b = label.new(bar_index + ext, price, txt + "  " + str.tostring(price, format.mintick),
             style = label.style_label_left, color = color.new(chart.bg_color, 100), textcolor = c, size = size.tiny)
        array.push(la, l)
        array.push(lb, b)
    true

// kotak sesi (kill zone / jendela SB)
f_session(string sess, string tz, string name, color col, int fillA, string bsty, bool show, int maxN, array<box> boxes) =>
    bool act = not na(time(timeframe.period, sess, tz))
    var float hi = na
    var float lo = na
    var box bx = na
    if show and act and not act[1]
        hi := high
        lo := low
        bx := box.new(bar_index, high, bar_index, low, border_color = color.new(col, 55),
             bgcolor = color.new(col, fillA), border_width = 1, border_style = bsty, text = name,
             text_size = size.tiny, text_color = color.new(col, 25),
             text_halign = text.align_left, text_valign = text.align_top)
        array.push(boxes, bx)
        if array.size(boxes) > maxN
            box.delete(array.shift(boxes))
    else if act and not na(bx)
        hi := math.max(hi, high)
        lo := math.min(lo, low)
        box.set_top(bx, hi)
        box.set_bottom(bx, lo)
        box.set_right(bx, bar_index)
    act

// high/low sesi terakhir yang sudah selesai (selama sesi berjalan = sesi sebelumnya)
f_sessRef(string sess, string tz) =>
    bool act = not na(time(timeframe.period, sess, tz))
    var float hi = na
    var float lo = na
    var float pHi = na
    var float pLo = na
    if act and not act[1]
        pHi := hi
        pLo := lo
        hi := high
        lo := low
    else if act
        hi := math.max(hi, high)
        lo := math.min(lo, low)
    [act ? pHi : hi, act ? pLo : lo]

f_candle(int x0, float o, float h, float l, float c, color body, color brd, array<box> bxs, array<line> lns) =>
    array.push(bxs, box.new(x0, math.max(o, c), x0 + 2, math.min(o, c), border_color = brd, bgcolor = body))
    array.push(lns, line.new(x0 + 1, h, x0 + 1, l, color = brd, width = 1))
    true

// 0 akumulasi | 1 bull (Judas turun lalu naik) | 2 bear (Judas naik lalu turun) | 3 dist naik | 4 dist turun
f_po3(float o, float h, float l, float c, float thr) =>
    float up = h - o
    float dn = o - l
    int r = 0
    if up > thr or dn > thr
        if dn > thr and c > o
            r := 1
        else if up > thr and c < o
            r := 2
        else if c > o
            r := 3
        else
            r := 4
    r

f_po3Txt(int r) =>
    r == 1 ? "BULL · Judas↓ → Dist↑" : r == 2 ? "BEAR · Judas↑ → Dist↓" : r == 3 ? "Dist ↑ (tanpa Judas)" : r == 4 ? "Dist ↓ (tanpa Judas)" : "AKUMULASI"

f_po3Short(int r) =>
    r == 1 ? "Judas↓ → Dist↑" : r == 2 ? "Judas↑ → Dist↓" : r == 3 ? "Dist ↑" : r == 4 ? "Dist ↓" : "Akumulasi"

// ---------------------------- DATA HTF -----------------------------
bool htf1Ok = timeframe.in_seconds(htf1) >= timeframe.in_seconds()
bool htf2Ok = timeframe.in_seconds(htf2) >= timeframe.in_seconds()

float pdh = request.security(syminfo.tickerid, "D", high[1], lookahead = barmerge.lookahead_on)
float pdl = request.security(syminfo.tickerid, "D", low[1],  lookahead = barmerge.lookahead_on)
float pwh = request.security(syminfo.tickerid, "W", high[1], lookahead = barmerge.lookahead_on)
float pwl = request.security(syminfo.tickerid, "W", low[1],  lookahead = barmerge.lookahead_on)
[o1, c1] = request.security(syminfo.tickerid, htf1, [open[1], close[1]], lookahead = barmerge.lookahead_on)
[o2, c2] = request.security(syminfo.tickerid, htf2, [open[1], close[1]], lookahead = barmerge.lookahead_on)

float dO  = request.security(syminfo.tickerid, "D", open, lookahead = barmerge.lookahead_on)
float wO  = request.security(syminfo.tickerid, "W", open, lookahead = barmerge.lookahead_on)
float mO  = request.security(syminfo.tickerid, "M", open, lookahead = barmerge.lookahead_on)
float h2O = request.security(syminfo.tickerid, htf2, open, lookahead = barmerge.lookahead_on)

int wTrend  = request.security(syminfo.tickerid, "W", f_trend(htfPivot))
int dTrend  = request.security(syminfo.tickerid, "D", f_trend(htfPivot))
int h1Trend = request.security(syminfo.tickerid, htf1, f_trend(htfPivot))
int h2Trend = request.security(syminfo.tickerid, htf2, f_trend(htfPivot))

// candle D1 kemarin + high/low hari ini (untuk SEQ GxT)
[dO1, dC1] = request.security(syminfo.tickerid, "D", [open[1], close[1]], lookahead = barmerge.lookahead_on)
float dATR  = request.security(syminfo.tickerid, "D", ta.atr(14)[1], lookahead = barmerge.lookahead_on)
float dHnow = request.security(syminfo.tickerid, "D", high, lookahead = barmerge.lookahead_on)
float dLnow = request.security(syminfo.tickerid, "D", low,  lookahead = barmerge.lookahead_on)

bool newD  = timeframe.change("D")
bool newW  = timeframe.change("W")
bool newM  = timeframe.change("M")
bool newH1 = timeframe.change(htf1)
bool newH2 = timeframe.change(htf2)
var int dStart  = 0
var int wStart  = 0
var int mStart  = 0
var int h1Start = 0
var int h2Start = 0
if newD
    dStart := bar_index
if newW
    wStart := bar_index
if newM
    mStart := bar_index
if newH1
    h1Start := bar_index
if newH2
    h2Start := bar_index

// ---------------------------- STRUKTUR -----------------------------
float ph  = ta.pivothigh(high, pivotLen, pivotLen)
float pl  = ta.pivotlow(low, pivotLen, pivotLen)
float atr = ta.atr(14)

var float sh = na
var int   shIdx = na
var bool  shBrk = false
var bool  shSwp = false
var float psh = na
var int   pshIdx = na

var float sl = na
var int   slIdx = na
var bool  slBrk = false
var bool  slSwp = false
var float psl = na
var int   pslIdx = na

var int trend = 0
var int lastSwpBar = -1000
var int lastSwpDir = 0

var array<line>  marksL   = array.new<line>()
var array<label> marksB   = array.new<label>()
var array<label> swingLbl = array.new<label>()
var array<label> sweepTag = array.new<label>()

if not na(ph)
    psh := sh
    pshIdx := shIdx
    sh := ph
    shIdx := bar_index - pivotLen
    shBrk := false
    shSwp := false
    if showSwing
        f_tag(shIdx, ph, na(psh) ? "H" : ph > psh ? "HH" : "LH", colLvl, label.style_label_down, swingLbl, 24)
    if showEQ and not na(psh) and math.abs(ph - psh) <= atr * eqTol
        f_mark(pshIdx, psh, shIdx, ph, "EQH", colBear, line.style_dotted,
             math.round((pshIdx + shIdx) / 2.0), math.max(psh, ph), label.style_label_down, marksL, marksB, maxMarks)

if not na(pl)
    psl := sl
    pslIdx := slIdx
    sl := pl
    slIdx := bar_index - pivotLen
    slBrk := false
    slSwp := false
    if showSwing
        f_tag(slIdx, pl, na(psl) ? "L" : pl > psl ? "HL" : "LL", colLvl, label.style_label_up, swingLbl, 24)
    if showEQ and not na(psl) and math.abs(pl - psl) <= atr * eqTol
        f_mark(pslIdx, psl, slIdx, pl, "EQL", colBull, line.style_dotted,
             math.round((pslIdx + slIdx) / 2.0), math.min(psl, pl), label.style_label_up, marksL, marksB, maxMarks)

bool bullBrk = barstate.isconfirmed and not na(sh) and not shBrk and close > sh
bool bearBrk = barstate.isconfirmed and not na(sl) and not slBrk and close < sl

if bullBrk
    bool mss = trend == -1
    if showBOS
        f_mark(shIdx, sh, bar_index, sh, mss ? "MSS" : "BOS", colBull, mss ? line.style_solid : line.style_dashed,
             math.round((shIdx + bar_index) / 2.0), sh, label.style_label_down, marksL, marksB, maxMarks)
    shBrk := true
    trend := 1

if bearBrk
    bool mss = trend == 1
    if showBOS
        f_mark(slIdx, sl, bar_index, sl, mss ? "MSS" : "BOS", colBear, mss ? line.style_solid : line.style_dashed,
             math.round((slIdx + bar_index) / 2.0), sl, label.style_label_up, marksL, marksB, maxMarks)
    slBrk := true
    trend := -1

// --------------------------- LIQUIDITY SWEEP -----------------------
bool bslSw = showSweeps and barstate.isconfirmed and not na(sh) and not shBrk and not shSwp and high > sh and close < sh
bool sslSw = showSweeps and barstate.isconfirmed and not na(sl) and not slBrk and not slSwp and low < sl and close > sl

if bslSw
    shSwp := true
    lastSwpBar := bar_index
    lastSwpDir := -1
    f_mark(shIdx, sh, bar_index, sh, "Sweep", colBear, line.style_dotted,
         bar_index, high, label.style_label_down, marksL, marksB, maxMarks)

if sslSw
    slSwp := true
    lastSwpBar := bar_index
    lastSwpDir := 1
    f_mark(slIdx, sl, bar_index, sl, "Sweep", colBull, line.style_dotted,
         bar_index, low, label.style_label_up, marksL, marksB, maxMarks)

var bool pdhSw = false
var bool pdlSw = false
if newD
    pdhSw := false
    pdlSw := false

if showSweeps and barstate.isconfirmed and not na(pdh) and not pdhSw and high > pdh and close < pdh
    pdhSw := true
    lastSwpBar := bar_index
    lastSwpDir := -1
    f_tag(bar_index, high, "PDH swept", colBear, label.style_label_down, sweepTag, 8)

if showSweeps and barstate.isconfirmed and not na(pdl) and not pdlSw and low < pdl and close > pdl
    pdlSw := true
    lastSwpBar := bar_index
    lastSwpDir := 1
    f_tag(bar_index, low, "PDL swept", colBull, label.style_label_up, sweepTag, 8)

// ---------------------------- DEALING RANGE ------------------------
float eqP = (not na(sh) and not na(sl) and sh > sl) ? math.avg(sh, sl) : na
string zoneTxt = na(eqP) ? "-" : close > eqP ? "PREMIUM" : close < eqP ? "DISCOUNT" : "EQ"

// ------------------------------ KILL ZONE --------------------------
var array<box> boxA = array.new<box>()
var array<box> boxL = array.new<box>()
var array<box> boxN = array.new<box>()
bool sessOk = showSess and timeframe.in_seconds() <= 3600
bool asiaAct   = f_session(asiaS,   sessTZ, "ASIA KZ",   color.gray,   92, line.style_solid, sessOk, maxSess, boxA)
bool londonAct = f_session(londonS, sessTZ, "LONDON KZ", color.orange, 92, line.style_solid, sessOk, maxSess, boxL)
bool nyAct     = f_session(nyS,     sessTZ, "NY KZ",     color.blue,   92, line.style_solid, sessOk, maxSess, boxN)
string sessTxt = asiaAct ? "ASIA KZ" : londonAct ? "LONDON KZ" : nyAct ? "NY KZ" : "OFF"

[aRefH, aRefL] = f_sessRef(asiaS,   sessTZ)
[lRefH, lRefL] = f_sessRef(londonS, sessTZ)
[nRefH, nRefL] = f_sessRef(nyS,     sessTZ)
float hiWin = ta.highest(high, sessRevWin)
float loWin = ta.lowest(low, sessRevWin)

// -------------------- SILVER BULLET + MACRO TIME (NY) --------------
string NY = "America/New_York"
var array<box> boxS1 = array.new<box>()
var array<box> boxS2 = array.new<box>()
var array<box> boxS3 = array.new<box>()
var array<box> boxS4 = array.new<box>()
var array<box> boxS5 = array.new<box>()
bool sbOk = showSBBox and timeframe.in_seconds() <= 3600
bool sbLdn = f_session(sbLdnS, NY, "SB LDN",    colMacro, 100, line.style_dashed, sbOk, maxSess, boxS1)
bool sbPre = f_session(sbPreS, NY, "SB NY PRE", colMacro, 100, line.style_dashed, sbOk, maxSess, boxS2)
bool sbAm  = f_session(sbAmS,  NY, "SB NY AM",  colMacro, 100, line.style_dashed, sbOk, maxSess, boxS3)
bool sbPm  = f_session(sbPmS,  NY, "SB NY PM",  colMacro, 100, line.style_dashed, sbOk, maxSess, boxS4)
bool sbFin = f_session(sbFinS, NY, "SB FIN",    colMacro, 100, line.style_dashed, sbOk, maxSess, boxS5)
bool sbAny = sbLdn or sbPre or sbAm or sbPm or sbFin
string sbName = sbLdn ? "SB LDN" : sbPre ? "SB NY PRE" : sbAm ? "SB NY AM" : sbPm ? "SB NY PM" : sbFin ? "SB FIN" : ""

int nyMin = minute(time, NY)

bool ictMacro = f_in("0750-0810", NY) or f_in("0850-0910", NY) or f_in("0950-1010", NY) or f_in("1050-1110", NY) or f_in("1150-1210", NY) or f_in("1320-1340", NY) or f_in("1450-1510", NY) or f_in("1515-1545", NY) or f_in("1550-1610", NY)
bool sbMacro  = sbAny and ((nyMin >= 20 and nyMin < 40) or nyMin >= 50 or nyMin < 10)
bool macroNow = (showICTMacro and ictMacro) or (showSBMacro and sbMacro)

bgcolor(macroNow ? color.new(colMacro, 93) : na, title = "Macro time")

// key times: open xx:15 dan xx:45 di dalam jendela SB
var array<line>  ktL = array.new<line>()
var array<label> ktB = array.new<label>()
bool k15 = sbAny and nyMin >= 15 and nz(nyMin[1], 99) < 15
bool k45 = sbAny and nyMin >= 45 and nz(nyMin[1], 99) < 45
int  nbKey = math.max(math.round(1500.0 / timeframe.in_seconds()), 1)

if showKeyTimes and (k15 or k45) and timeframe.in_seconds() <= 900
    line kl = line.new(bar_index, open, bar_index + nbKey, open, color = colMacro, style = line.style_dotted)
    label kb = label.new(bar_index + nbKey, open, (k15 ? "xx:15" : "xx:45") + " open", style = label.style_label_left,
         color = color.new(chart.bg_color, 100), textcolor = colMacro, size = size.tiny)
    array.push(ktL, kl)
    array.push(ktB, kb)
    if array.size(ktL) > 10
        line.delete(array.shift(ktL))
        label.delete(array.shift(ktB))

string timeTxt = (macroNow ? "MACRO · " : "") + (sbName != "" ? sbName : sessTxt)

// ------------------------------ FVG / IFVG -------------------------
var array<box>   fzBox = array.new<box>()
var array<float> fzTop = array.new<float>()
var array<float> fzBot = array.new<float>()
var array<int>   fzDir = array.new<int>()
var array<int>   fzSt  = array.new<int>()
var array<label> ifvgTag = array.new<label>()

var string lastIfvgTxt = "-"
var int    lastIfvgDir = 0
bool evBull = false
bool evBear = false

bool newBullFVG = barstate.isconfirmed and low > high[2] and (low - high[2]) > atr * fvgMin and close[1] > open[1]
bool newBearFVG = barstate.isconfirmed and high < low[2] and (low[2] - high) > atr * fvgMin and close[1] < open[1]

if newBullFVG or newBearFVG
    bool bu = newBullFVG
    float zt = bu ? low : low[2]
    float zb = bu ? high[2] : high
    color zc = bu ? colBull : colBear
    box nb = box.new(bar_index - 2, zt, bar_index, zb, border_color = color.new(zc, 100),
         bgcolor = color.new(zc, showFVG ? 88 : 100))
    array.push(fzBox, nb)
    array.push(fzTop, zt)
    array.push(fzBot, zb)
    array.push(fzDir, bu ? 1 : -1)
    array.push(fzSt, 0)
    if array.size(fzBox) > maxFVG
        box.delete(array.shift(fzBox))
        array.shift(fzTop)
        array.shift(fzBot)
        array.shift(fzDir)
        array.shift(fzSt)

if array.size(fzBox) > 0
    for i = array.size(fzBox) - 1 to 0
        box zx = array.get(fzBox, i)
        float tp = array.get(fzTop, i)
        float bt = array.get(fzBot, i)
        int dr = array.get(fzDir, i)
        int st = array.get(fzSt, i)
        box.set_right(zx, bar_index)
        bool rm = false
        if barstate.isconfirmed
            if st == 0
                bool invUp = dr == -1 and close > tp
                bool invDn = dr == 1 and close < bt
                if invUp or invDn
                    int invDir = invUp ? 1 : -1
                    bool cM = macroNow
                    bool cS = lastSwpDir == invDir and bar_index - lastSwpBar <= sweepWin
                    bool cD = math.abs(close - open) >= atr * dispMin
                    bool cA = (h2Trend == invDir or dTrend == invDir) and (invDir == 1 ? zoneTxt == "DISCOUNT" : zoneTxt == "PREMIUM")
                    int score = (cM ? 1 : 0) + (cS ? 1 : 0) + (cD ? 1 : 0) + (cA ? 1 : 0)
                    array.set(fzSt, i, 1)
                    box.set_bgcolor(zx, color.new(color.gray, showIFVG ? 75 : 100))
                    box.set_border_color(zx, color.new(color.gray, showIFVG ? 35 : 100))
                    if showIFVG
                        f_tag(bar_index, invDir == 1 ? low : high, "IFVG" + (invDir == 1 ? "↑ " : "↓ ") + str.tostring(score) + "/4",
                             invDir == 1 ? colBull : colBear, invDir == 1 ? label.style_label_up : label.style_label_down, ifvgTag, 10)
                    lastIfvgDir := invDir
                    lastIfvgTxt := (invDir == 1 ? "BULL " : "BEAR ") + str.tostring(score) + "/4 · " + (cM ? "M" : "-") + (cS ? "S" : "-") + (cD ? "D" : "-") + (cA ? "A" : "-")
                    if invDir == 1
                        evBull := true
                    else
                        evBear := true
            else
                if (dr == 1 and close > tp) or (dr == -1 and close < bt)
                    rm := true
        if rm
            box.delete(zx)
            array.remove(fzBox, i)
            array.remove(fzTop, i)
            array.remove(fzBot, i)
            array.remove(fzDir, i)
            array.remove(fzSt, i)

alertcondition(evBull, "IFVG Bullish", "Inversi FVG bullish terdeteksi")
alertcondition(evBear, "IFVG Bearish", "Inversi FVG bearish terdeteksi")

// ------------------------- OPEN BERDASARKAN WAKTU ------------------
var array<int>    tMin = array.new<int>()
var array<string> tLbl = array.new<string>()
var array<float>  tPx  = array.new<float>()
var array<int>    tIdx = array.new<int>()

if barstate.isfirst
    array<string> rows = str.split(openList, "\n")
    for r in rows
        string t = str.trim(r)
        if str.length(t) > 0 and array.size(tMin) < 8
            array<string> p = str.split(t, ",")
            string hhmm = str.trim(array.get(p, 0))
            if str.length(hhmm) == 4
                float hh = str.tonumber(str.substring(hhmm, 0, 2))
                float mm = str.tonumber(str.substring(hhmm, 2, 4))
                if not na(hh) and not na(mm)
                    array.push(tMin, int(hh) * 60 + int(mm))
                    array.push(tLbl, array.size(p) > 1 ? str.trim(array.get(p, 1)) : hhmm)
                    array.push(tPx, na)
                    array.push(tIdx, -1)

int  curMin   = hour(time, openTZ) * 60 + minute(time, openTZ)
bool newDayTz = ta.change(dayofmonth(time, openTZ)) != 0
bool openOk   = showTimeOpens and timeframe.in_seconds() <= 3600

if openOk and array.size(tMin) > 0
    for i = 0 to array.size(tMin) - 1
        int tg = array.get(tMin, i)
        if curMin >= tg and (nz(curMin[1], -1) < tg or newDayTz)
            array.set(tPx, i, open)
            array.set(tIdx, i, bar_index)

// ---------------------------------- PO3 ----------------------------
string po3Auto_tf = switch timeframe.period
    "1"   => "15"
    "3"   => "30"
    "5"   => "60"
    "15"  => "240"
    "30"  => "240"
    "60"  => "D"
    "240" => "W"
    "D"   => "M"
    => po3Manual
string po3TF = po3Auto ? po3Auto_tf : po3Manual
bool   po3Ok = timeframe.in_seconds(po3TF) > timeframe.in_seconds()

[q0o, q0h, q0l, q0c, q1o, q1h, q1l, q1c, q2o, q2h, q2l, q2c, q3o, q3h, q3l, q3c, qATR] = request.security(
     syminfo.tickerid, po3TF,
     [open, high, low, close, open[1], high[1], low[1], close[1], open[2], high[2], low[2], close[2], open[3], high[3], low[3], close[3], ta.atr(14)],
     lookahead = barmerge.lookahead_on)

bool newP = timeframe.change(po3TF)
var int po3Start = 0
if newP
    po3Start := bar_index

int po3CloseT = time_close(po3TF)
int po3Code   = f_po3(q0o, q0h, q0l, q0c, nz(qATR) * po3Thr)
color po3Col  = (po3Code == 1 or po3Code == 3) ? colBull : (po3Code == 2 or po3Code == 4) ? colBear : colLvl

// ------------------ GAMBAR LEVEL (hanya di bar terakhir) -----------
var array<line>  lvL = array.new<line>()
var array<label> lvB = array.new<label>()
var array<box>   bxs = array.new<box>()

if barstate.islast
    for l in lvL
        line.delete(l)
    for b in lvB
        label.delete(b)
    for x in bxs
        box.delete(x)
    array.clear(lvL)
    array.clear(lvB)
    array.clear(bxs)

    // --- level HTF
    if showPDH
        f_level(pdh, dStart, "PDH", colLvl, line.style_solid, lvL, lvB, lineExt)
        f_level(pdl, dStart, "PDL", colLvl, line.style_solid, lvL, lvB, lineExt)
    if showPWH
        f_level(pwh, wStart, "PWH", colLvl, line.style_dashed, lvL, lvB, lineExt)
        f_level(pwl, wStart, "PWL", colLvl, line.style_dashed, lvL, lvB, lineExt)
    if showOCL2 and htf2Ok
        f_level(o2, h2Start, f_tf(htf2) + " Open",  colLvl, line.style_dotted, lvL, lvB, lineExt)
        f_level(c2, h2Start, f_tf(htf2) + " Close", colLvl, line.style_dotted, lvL, lvB, lineExt)
    if showOCL1 and htf1Ok
        f_level(o1, h1Start, f_tf(htf1) + " Open",  colLvl, line.style_dotted, lvL, lvB, lineExt)
        f_level(c1, h1Start, f_tf(htf1) + " Close", colLvl, line.style_dotted, lvL, lvB, lineExt)

    // --- opening prices (candle berjalan)
    if showDO
        f_level(dO, dStart, "D Open", colOpen, line.style_dotted, lvL, lvB, lineExt)
    if showWO
        f_level(wO, wStart, "W Open", colOpen, line.style_dotted, lvL, lvB, lineExt)
    if showMO
        f_level(mO, mStart, "M Open", colOpen, line.style_dotted, lvL, lvB, lineExt)
    if showH2O and htf2Ok and not (showPO3 and po3Ok and po3TF == htf2)
        f_level(h2O, h2Start, f_tf(htf2) + " Open", colOpen, line.style_dotted, lvL, lvB, lineExt)

    // --- opening prices (berdasarkan waktu)
    if openOk and array.size(tMin) > 0
        for i = 0 to array.size(tMin) - 1
            if array.get(tIdx, i) >= 0
                f_level(array.get(tPx, i), array.get(tIdx, i), array.get(tLbl, i) + " Open", colOpen, line.style_dotted, lvL, lvB, lineExt)

    // --- dealing range + OTE
    if not na(eqP)
        int left = math.max(math.min(shIdx, slIdx), bar_index - 4500)
        if showPD
            array.push(bxs, box.new(left, sh, bar_index + lineExt, eqP,
                 border_color = color.new(colBear, 100), bgcolor = color.new(colBear, 93)))
            array.push(bxs, box.new(left, eqP, bar_index + lineExt, sl,
                 border_color = color.new(colBull, 100), bgcolor = color.new(colBull, 93)))
            f_level(eqP, left, "EQ 50%", colLvl, line.style_dashed, lvL, lvB, lineExt)
        if showOTE
            float rng = sh - sl
            bool upLeg = slIdx < shIdx
            float oA = upLeg ? sh - 0.62 * rng : sl + 0.62 * rng
            float oB = upLeg ? sh - 0.79 * rng : sl + 0.79 * rng
            float oI = upLeg ? sh - 0.705 * rng : sl + 0.705 * rng
            array.push(bxs, box.new(left, math.max(oA, oB), bar_index + lineExt, math.min(oA, oB),
                 border_color = color.new(#2962ff, 100), bgcolor = color.new(#2962ff, 90)))
            f_level(oI, left, "OTE 0.705", #2962ff, line.style_dotted, lvL, lvB, lineExt)

    // --- PO3 candles
    if showPO3 and po3Ok
        int base = bar_index + po3Offset
        color brd = color.new(chart.fg_color, 35)
        bool swUp = q1h > q2h and q1c < q2h
        bool swDn = q1l < q2l and q1c > q2l

        f_candle(base,     q3o, q3h, q3l, q3c, q3c > q3o ? color.new(colBull, 25) : color.new(colBear, 25), brd, bxs, lvL)
        f_candle(base + 3, q2o, q2h, q2l, q2c, q2c > q2o ? color.new(colBull, 25) : color.new(colBear, 25), brd, bxs, lvL)
        f_candle(base + 6, q1o, q1h, q1l, q1c, (swUp or swDn) ? color.new(color.orange, 20) : (q1c > q1o ? color.new(colBull, 25) : color.new(colBear, 25)), brd, bxs, lvL)
        f_candle(base + 9, q0o, q0h, q0l, q0c, q0c > q0o ? color.new(colBull, 25) : color.new(colBear, 25), brd, bxs, lvL)

        float yTop = math.max(math.max(q0h, q1h), math.max(q2h, q3h))
        f_lbl(base + 5, yTop, "PO3 · " + f_tf(po3TF), colLvl, label.style_label_down, lvB)
        if swUp or swDn
            f_lbl(base + 7, swUp ? q1h : q1l, "Sweep", color.orange, swUp ? label.style_label_down : label.style_label_up, lvB)

        f_level(q0o, po3Start, f_tf(po3TF) + " Open", colOpen, line.style_solid, lvL, lvB, lineExt)

        float midT = q1c > q1o ? (q1h + q1c) / 2 : (q1h + q1o) / 2
        float midB = q1c > q1o ? (q1l + q1o) / 2 : (q1l + q1c) / 2
        if showMidWick
            array.push(lvL, line.new(base + 7, midT, base + 11, midT, color = colLvl, style = line.style_dashed))
            array.push(lvL, line.new(base + 7, midB, base + 11, midB, color = colLvl, style = line.style_dashed))
            if projMidWick
                f_level(midT, po3Start, "0.5W▲", colLvl, line.style_dotted, lvL, lvB, lineExt)
                f_level(midB, po3Start, "0.5W▼", colLvl, line.style_dotted, lvL, lvB, lineExt)
        if showCE
            float ceP = (q1h + q1l) / 2
            array.push(lvL, line.new(base + 7, ceP, base + 11, ceP, color = colLvl, style = line.style_dotted))
            if projMidWick
                f_level(ceP, po3Start, "CE", colLvl, line.style_dotted, lvL, lvB, lineExt)

        f_lbl(base + 12, q0c, f_po3Short(po3Code), po3Col, label.style_label_left, lvB)

        if showCount and not na(po3CloseT)
            int bl = math.min(math.max(math.round((po3CloseT - time) / (timeframe.in_seconds() * 1000.0)), 0), 480)
            array.push(lvL, line.new(bar_index + bl, low, bar_index + bl, high, extend = extend.both,
                 color = color.new(colLvl, 40), style = line.style_dotted, width = 1))
            int rem = math.max(po3CloseT - timenow, 0)
            int rs  = int(rem / 1000)
            string cd = str.tostring(int(rs / 3600), "00") + ":" + str.tostring(int((rs % 3600) / 60), "00") + ":" + str.tostring(rs % 60, "00")
            f_lbl(bar_index + bl, high, "close " + cd, colLvl, label.style_label_down, lvB)

// ------------------------------ DASHBOARD --------------------------
var table dash = table.new(f_pos(dashPos), 2, 11, bgcolor = color.new(chart.bg_color, 15), border_width = 0)

f_cell(int col, int row, string txt, color tc) =>
    table.cell(dash, col, row, txt, text_color = tc, text_size = size.tiny, text_halign = text.align_left,
         bgcolor = color.new(chart.bg_color, 15))
    true

f_tcol(int t) => t == 1 ? colBull : t == -1 ? colBear : chart.fg_color

// ---- SEQ GxT: Reversal / Continue / Aligned
bool allUp = wTrend == 1 and dTrend == 1 and h2Trend == 1 and h1Trend == 1 and trend == 1
bool allDn = wTrend == -1 and dTrend == -1 and h2Trend == -1 and h1Trend == -1 and trend == -1

float dMid = math.avg(pdh, pdl)
bool dExpBear = dC1 < dO1 and (dO1 - dC1) >= nz(dATR) * seqExp
bool dExpBull = dC1 > dO1 and (dC1 - dO1) >= nz(dATR) * seqExp

bool dHigh = dHnow > pdh and close < pdh
bool wHigh = dHnow > pwh and close < pwh
bool aHigh = sessOk and hiWin > aRefH and close < aRefH
bool lHigh = sessOk and hiWin > lRefH and close < lRefH
bool nHigh = sessOk and hiWin > nRefH and close < nRefH
bool dLow = dLnow < pdl and close > pdl
bool wLow = dLnow < pwl and close > pwl
bool aLow = sessOk and loWin < aRefL and close > aRefL
bool lLow = sessOk and loWin < lRefL and close > lRefL
bool nLow = sessOk and loWin < nRefL and close > nRefL

bool rD = dHigh or wHigh or aHigh or lHigh or nHigh
bool rU = dLow or wLow or aLow or lLow or nLow
bool revDn = rD and not rU
bool revUp = rU and not rD
string revSrc = revDn ? (dHigh ? "PDH" : wHigh ? "PWH" : aHigh ? "Asia H" : lHigh ? "London H" : "NY H") : revUp ? (dLow ? "PDL" : wLow ? "PWL" : aLow ? "Asia L" : lLow ? "London L" : "NY L") : ""

bool contDn = dExpBear and close < dMid
bool contUp = dExpBull and close > dMid

string seqTxt = revDn ? "REVERSAL ↓ · " + revSrc : revUp ? "REVERSAL ↑ · " + revSrc : contDn ? "CONTINUE ↓" : contUp ? "CONTINUE ↑" : allUp ? "ALIGNED ↑" : allDn ? "ALIGNED ↓" : "MIXED"
bool seqUp = revUp or contUp or (allUp and not revDn and not contDn)
bool seqDn = revDn or contDn or (allDn and not revUp and not contUp)
string profTxt = asiaAct ? "ASIA · protraksi" : londonAct ? "LONDON · reversal" : nyAct ? "NY · continue" : "-"

if barstate.islast and showDash
    f_cell(0, 0, "W1", chart.fg_color)
    f_cell(1, 0, f_txt(wTrend), f_tcol(wTrend))
    f_cell(0, 1, "D1", chart.fg_color)
    f_cell(1, 1, f_txt(dTrend), f_tcol(dTrend))
    f_cell(0, 2, f_tf(htf2), chart.fg_color)
    f_cell(1, 2, htf2Ok ? f_txt(h2Trend) : "-", htf2Ok ? f_tcol(h2Trend) : chart.fg_color)
    f_cell(0, 3, f_tf(htf1), chart.fg_color)
    f_cell(1, 3, htf1Ok ? f_txt(h1Trend) : "-", htf1Ok ? f_tcol(h1Trend) : chart.fg_color)
    f_cell(0, 4, "LTF " + timeframe.period, chart.fg_color)
    f_cell(1, 4, f_txt(trend), f_tcol(trend))
    f_cell(0, 5, "SEQ", chart.fg_color)
    f_cell(1, 5, seqTxt, seqUp ? colBull : seqDn ? colBear : chart.fg_color)
    f_cell(0, 6, "ZONE", chart.fg_color)
    f_cell(1, 6, zoneTxt, zoneTxt == "PREMIUM" ? colBear : zoneTxt == "DISCOUNT" ? colBull : chart.fg_color)
    f_cell(0, 7, "TIME", chart.fg_color)
    f_cell(1, 7, timeTxt, macroNow ? colMacro : chart.fg_color)
    f_cell(0, 8, "PO3 " + f_tf(po3TF), chart.fg_color)
    f_cell(1, 8, po3Ok ? f_po3Txt(po3Code) : "-", po3Ok ? po3Col : chart.fg_color)
    f_cell(0, 9, "IFVG", chart.fg_color)
    f_cell(1, 9, lastIfvgTxt, lastIfvgDir == 1 ? colBull : lastIfvgDir == -1 ? colBear : chart.fg_color)
    f_cell(0, 10, "PROFILE", chart.fg_color)
    f_cell(1, 10, profTxt, chart.fg_color)
