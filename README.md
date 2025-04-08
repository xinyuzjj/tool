1.Breaker Blocks with Signals [LuxAlgo]
// This work is licensed under a Attribution-NonCommercial-ShareAlike 4.0 International (CC Dy-NC-SA 4.0) https://creativecommons.org/licenses/Dy-nc-sa/4.0/
// © LuxAlgo

//@version=5
indicator("Breaker Blocks with Signals [LuxAlgo]", max_lines_count=500, max_boxes_count=500, max_labels_count=500, max_bars_back=3000, overlay=true)
//------------------------------------------------------------------------------
//Settings
//-----------------------------------------------------------------------------{
shZZ                  = false 
len                   = input.int   (    5     , title='      Length'              , inline='MS' , group='Market Structure' 
                      ,                                         minval=1, maxval=10                                                                      )
//Breaker block
breakerCandleOnlyBody = input.bool  (  false   , title='Use only candle body'                    , group='Breaker Block'                                 )
breakerCandle_2Last   = input.bool  (  false   , title='Use 2 candles instead of 1'              , group='Breaker Block', tooltip='In the same direction')
tillFirstBreak        = input.bool  (  true   , title='Stop at first break of center line'      , group='Breaker Block'                                 )

//PD array
onlyWhenInPDarray     = input.bool  (  false   , title='Only when E is in Premium/Discount Array', group='PD array'                                      )
showPDarray           = input.bool  (  false   , title='show Premium/Discount Zone'              , group='PD array'                                      )
showBreaks            = input.bool  (  false   , title='Highlight Swing Breaks'                  , group='PD array'                                      )
showSPD               = input.bool  (  true    , title='Show Swings/PD Arrays'                   , group='PD array'                                      )
PDtxtCss              = input.color (  color.silver, 'Text Color'      , group='PD array'                                      )
PDSCss                = input.color (  color.silver, 'Swing Line Color', group='PD array'                                      )

//Take profit
iTP                   = input.bool  (  false   , title='Enable TP'                 , inline='RR' , group='TP'                                            )
tpCss                 = input.color ( #2157f3, title=''                  , inline='RR', group='TP'                                            )
R1a                   = input.float (    1     , title='R:R 1', minval= 1, maxval=1, inline='RR1', group='TP'                                            )
R2a                   = input.float (    2     , title= ':'   , minval=.2, step= .1, inline='RR1', group='TP'                                            )
R1b                   = input.float (    1     , title='R:R 2', minval= 1, maxval=1, inline='RR2', group='TP'                                            )
R2b                   = input.float (    3     , title= ':'   , minval=.2, step= .1, inline='RR2', group='TP'                                            )
R1c                   = input.float (    1     , title='R:R 3', minval= 1, maxval=1, inline='RR3', group='TP'                                            )
R2c                   = input.float (    4     , title= ':'   , minval=.2, step= .1, inline='RR3', group='TP'                                            )
 
//Colors
cBBplusA              = input.color (color.rgb(12, 181, 26, 93)
                                               , title='      '                    , inline='bl' , group='Colours    +BB                   Last Swings'  )
cBBplusB              = input.color (color.rgb(12, 181, 26, 85)
                                               , title=''                          , inline='bl' , group='Colours    +BB                   Last Swings'  )
cSwingBl              = input.color (color.rgb(255, 82, 82, 85)
                                               , title='        '                  , inline='bl' , group='Colours    +BB                   Last Swings'  )
cBB_minA              = input.color (color.rgb(255, 17, 0, 95)
                                               , title='      '                    , inline='br' , group='Colours    -BB                   Last Swings'  )
cBB_minB              = input.color (color.rgb(255, 17, 0, 85)
                                               , title=''                          , inline='br' , group='Colours    -BB                   Last Swings'  )
cSwingBr              = input.color (color.rgb(0, 137, 123, 85)
                                               , title='        '                  , inline='br' , group='Colours    -BB                   Last Swings'  )

_arrowup = '▲'
_arrowdn = '▼'
_c = '●'
_x = '❌'

//-----------------------------------------------------------------------------}
//General Calculations
//-----------------------------------------------------------------------------{
per        = last_bar_index - bar_index <= 2000 
mx         = math.max(close , open     )
mn         = math.min(close , open )
atr        = ta.atr  (10    )
n          = bar_index
hi         = high  
lo         = low   
mCxSize    = 50

//-----------------------------------------------------------------------------}
//User Defined Types
//-----------------------------------------------------------------------------{
type ZZ 
    int   [] d
    int   [] x 
    float [] y 
    line  [] l
    bool  [] b

type mss 
    int     dir
    line [] l_mssBl
    line [] l_mssBr
    line [] l_bosBl
    line [] l_bosBr
    label[] lbMssBl
    label[] lbMssBr
    label[] lbBosBl
    label[] lbBosBr

type block
    int   dir
    bool  Broken
    bool  Mitigated
    box   BB_boxA
    box   BB_boxB
    line  BB_line
    box   FVG_box
    line  line_1
    line  line_2
    bool  Broken_1
    bool  Broken_2
    box   PDa_boxA
    box   PDa_boxB
    box   PDa_box1
    line  PDaLine1
    label PDaLab_1
    box   PDa_box2
    line  PDaLine2    
    label PDaLab_2
    bool  PDbroken1
    bool  PDbroken2
    line  TP1_line
    line  TP2_line
    line  TP3_line
    bool  TP1_hit
    bool  TP2_hit
    bool  TP3_hit
    bool  scalp
    label HL
    label[] aLabels

//-----------------------------------------------------------------------------}
//Variables
//-----------------------------------------------------------------------------{
BBplus = 0, signUP = 1, cnclUP = 2, LL1break = 3, LL2break = 4, SW1breakUP = 5
 ,      SW2breakUP = 6,  tpUP1 = 7,    tpUP2 = 8,    tpUP3 = 9,   BB_endBl =10
BB_min =11, signDN =12, cnclDN =13, HH1break =14, HH2break =15, SW1breakDN =16
 ,      SW2breakDN =17,  tpDN1 =18,    tpDN2 =19,    tpDN3 =20,   BB_endBr =21

signals = 
 array.from(
   false // BBplus
 , false // signUP
 , false // cnclUP
 , false // LL1break
 , false // LL2break
 , false // SW1breakUP
 , false // SW2breakUP
 , false // tpUP1
 , false // tpUP2
 , false // tpUP3
 , false // BB_endBl
 , false // BB_min
 , false // signDN
 , false // cnclDN
 , false // HH1break
 , false // HH2break
 , false // SW1breakDN
 , false // SW2breakDN
 , false // tpDN1
 , false // tpDN2
 , false // tpDN3
 , false // BB_endBr
 )

var block   [] aBlockBl   = array.new<   block  >(          )
var block   [] aBlockBr   = array.new<   block  >(          )

var  ZZ         aZZ       = 
 ZZ.new(
 array.new < int    >(mCxSize,  0), 
 array.new < int    >(mCxSize,  0), 
 array.new < float  >(mCxSize, na),
 array.new < line   >(mCxSize, na),
 array.new < bool   >(mCxSize, na))

var mss MSS = mss.new(
 0
 , array.new < line  >()
 , array.new < line  >() 
 , array.new < line  >()
 , array.new < line  >()
 , array.new < label >() 
 , array.new < label >()
 , array.new < label >()
 , array.new < label >()
 )

var block BB = block.new(
   BB_boxA   = box.new  (na, na, na, na, border_color=color(na))
 , BB_boxB   = box.new  (na, na, na, na, border_color=color(na)
  , text_size=size.small
  , text_halign=text.align_right
  , text_font_family=font.family_monospace
  )
 , BB_line   = line.new (na, na, na, na
  , style=line.style_dashed
  , color=color.silver
  )
 , PDa_box1  = box.new  (na, na, na, na, border_color=color(na))
 , PDaLine1  = line.new (na, na, na, na, color=PDSCss)
 , PDaLab_1  = label.new(na, na, color=color(na))
 , PDa_box2  = box.new  (na, na, na, na, border_color=color(na))
 , PDaLine2  = line.new (na, na, na, na, color=PDSCss)
 , PDaLab_2  = label.new(na, na, color=color(na))
 , line_1    = line.new (na, na, na, na, color=PDSCss)
 , line_2    = line.new (na, na, na, na, color=PDSCss)
 , TP1_line  = line.new (na, na, na, na, color=tpCss)
 , TP2_line  = line.new (na, na, na, na, color=tpCss)
 , TP3_line  = line.new (na, na, na, na, color=tpCss)
 , HL        = label.new(na, na, color=color(na)
  , textcolor=PDtxtCss
  , yloc=yloc.price
  )
 , aLabels   = array.new<label>(1, label(na))
 )

//-----------------------------------------------------------------------------}
//Functions/methods
//-----------------------------------------------------------------------------{
method in_out(ZZ aZZ, int d, int x1, float y1, int x2, float y2, color col, bool b) =>
    aZZ.d.unshift(d), aZZ.x.unshift(x2), aZZ.y.unshift(y2), aZZ.b.unshift(b), aZZ.d.pop(), aZZ.x.pop(), aZZ.y.pop(), aZZ.b.pop()
    if shZZ
        aZZ.l.unshift(line.new(x1, y1, x2, y2, color= col)), aZZ.l.pop().delete()

method io_box(box[] aB, box b) => aB.unshift(b), aB.pop().delete()

method setLine(line ln, int x1, float y1, int x2, float y2) => ln.set_xy1(x1, y1), ln.set_xy2(x2, y2)

method notransp(color css) => color.rgb(color.r(css), color.g(css), color.b(css))

createLab(string s, float y, color c, string t, string sz = size.small) =>      
    label.new(n
     , y
     , style=s == 'c' ? label.style_label_center    
      : s == 'u' ? label.style_label_up 
      : label.style_label_down
     , textcolor=c
     , color=color(na)    
     , size=sz
     , text=t
     )

draw(left, col) =>
    
    max_bars_back(time, 1000)
    var int dir= na, var int x1= na, var float y1= na, var int x2= na, var float y2= na
    
    sz       = aZZ.d.size( )
    x2      := bar_index -1
    ph       = ta.pivothigh(hi, left, 1)
    pl       = ta.pivotlow (lo, left, 1)
    if ph   
        dir := aZZ.d.get (0) 
        x1  := aZZ.x.get (0) 
        y1  := aZZ.y.get (0) 
        y2  :=      nz(hi[1])
        
        if dir <  1  // if previous point was a pl, add, and change direction ( 1)
            aZZ.in_out( 1, x1, y1, x2, y2, col, true)
        else
            if dir ==  1 and ph > y1 
                aZZ.x.set(0, x2), aZZ.y.set(0, y2)   
                if shZZ
                    aZZ.l.get(0).set_xy2(x2, y2)        

    if pl
        dir := aZZ.d.get (0) 
        x1  := aZZ.x.get (0) 
        y1  := aZZ.y.get (0) 
        y2  :=      nz(lo[1])
        
        if dir > -1  // if previous point was a ph, add, and change direction (-1)
            aZZ.in_out(-1, x1, y1, x2, y2, col, true)
        else
            if dir == -1 and pl < y1 
                aZZ.x.set(0, x2), aZZ.y.set(0, y2)       
                if shZZ
                    aZZ.l.get(0).set_xy2(x2, y2)   
    
    iH = aZZ.d.get(2) ==  1 ? 2 : 1
    iL = aZZ.d.get(2) == -1 ? 2 : 1
    
    switch
        // MSS Bullish
        close > aZZ.y.get(iH) and aZZ.d.get(iH) ==  1 and MSS.dir <  1 and per =>
            
            Ex   = aZZ.x.get(iH -1), Ey = aZZ.y.get(iH -1) 
            Dx   = aZZ.x.get(iH   ), Dy = aZZ.y.get(iH   ), DyMx = mx[n - Dx]
            Cx   = aZZ.x.get(iH +1), Cy = aZZ.y.get(iH +1) 
            Bx   = aZZ.x.get(iH +2), By = aZZ.y.get(iH +2), ByMx = mx[n - Bx] 
            Ax   = aZZ.x.get(iH +3), Ay = aZZ.y.get(iH +3), AyMn = mn[n - Ax]
            _y   = math.max(ByMx, DyMx)
            mid  = AyMn + ((_y - AyMn) / 2) // 50% fib A- min(B, D)
            isOK = onlyWhenInPDarray ? Ay < Cy and Ay < Ey and Ey < mid : true
            
            float green1prT = na
            float green1prB = na
            float    avg    = na

            if Ey < Cy and Cx != Dx and isOK 
                // latest HH to 1 HH further -> search first green bar
                for i = n - Dx to n - Cx
                    if close[i] > open[i]
                        // reset previous swing box's
                        BB.PDa_box1.set_lefttop(na, na), BB.PDaLine1.set_xy1(na, na), BB.PDaLab_1.set_xy(na, na)
                        BB.PDa_box2.set_lefttop(na, na), BB.PDaLine2.set_xy1(na, na), BB.PDaLab_2.set_xy(na, na)
                        
                        green1idx   = n - i
                        green1prT  := breakerCandleOnlyBody ? mx[i] : high[i]
                        green1prB  := breakerCandleOnlyBody ? mn[i] : low [i]
                        if breakerCandle_2Last 
                            if close[i +1] > open[i +1]
                                green2prT  = breakerCandleOnlyBody ? mx[i +1] : high[i +1]
                                green2prB  = breakerCandleOnlyBody ? mn[i +1] : low [i +1]
                                if green2prT > green1prT or green2prB < green1prB
                                    green1idx -= 1
                                green1prT := math.max(green1prT, green2prT)
                                green1prB := math.min(green1prB, green2prB)
                        
                        // Breaker Block + 
                        avg := math.avg(green1prB, green1prT)
                        while BB.aLabels.size() > 0
                            BB.aLabels.pop().delete()
                        BB.PDa_boxA.delete(), BB.PDa_boxB.delete(), BB.dir :=  1
                        BB.BB_boxA.set_left   (green1idx)
                        BB.BB_boxA.set_top    (green1prT)
                        BB.BB_boxA.set_right  (    n    )
                        BB.BB_boxA.set_bottom (green1prB)
                        BB.BB_boxA.set_bgcolor(cBBplusA )

                        BB.BB_boxB.set_left   (    n    )
                        BB.BB_boxB.set_top    (green1prT)
                        BB.BB_boxB.set_right  (    n + 8)
                        BB.BB_boxB.set_bottom (green1prB)
                        BB.BB_boxB.set_bgcolor(cBBplusB )
                        BB.BB_boxB.set_text('+BB')
                        BB.BB_boxB.set_text_color(cBBplusB.notransp())
                        BB.BB_boxB.set_text_valign(text.align_bottom)
                       
                        BB.BB_line.set_xy1(n, avg), BB.BB_line.set_xy2(n , avg)

                        if showSPD
                            BB.line_1.set_xy1(Cx, Cy), BB.line_1.set_xy2(n , Cy), BB.Broken_1 := false
                            BB.line_2.set_xy1(Ex, Ey), BB.line_2.set_xy2(n , Ey), BB.Broken_2 := false
                            BB.HL.set_xy(Ex, Ey), BB.HL.set_style(label.style_label_up), BB.HL.set_text('LL')
                        
                        BB.TP1_hit    := false     
                        BB.TP2_hit    := false                              
                        BB.TP3_hit    := false  
                        BB.Broken     := false
                        BB.Mitigated  := false
                        BB.scalp      := false
                        BB.PDbroken1  := false
                        BB.PDbroken2  := false

                        if onlyWhenInPDarray and showPDarray
                            BB.PDa_boxA := box.new(Ax, mid, Ex +1, AyMn, bgcolor=color.rgb(132, 248, 171, 90), border_color=color(na)
                             , text = 'Discount PD Array', text_size = size.small, text_color = color.rgb(132, 248, 171, 25)
                             , text_halign = text.align_right, text_valign = text.align_center, text_font_family = font.family_monospace) // , text_wrap= text.wrap_auto
                            BB.PDa_boxB := box.new(Ax,  _y, Ex +1,  mid, bgcolor=color.rgb(248, 153, 132, 90), border_color=color(na))
                        
                        // Previous swings
                        cnt = 0, hh1 = high
                        for c = 0 to sz -2
                            getX = aZZ.x.get(c)
                            getY = aZZ.y.get(c)
                            if getY > hh1 and aZZ.d.get(c) ==  1 and showSPD
                                getY2 = (high[n - getX] - mn[n - getX]) / 4
                                switch cnt
                                    0 =>
                                        BB.PDa_box1.set_lefttop    (getX,        getY )
                                        BB.PDaLine1.set_xy1        (getX,        getY )
                                        BB.PDa_box1.set_rightbottom(n   , getY - getY2)
                                        BB.PDaLine1.set_xy2        (n   , getY        )
                                        BB.PDa_box1.set_bgcolor    (       cSwingBl   )
                                        BB.PDaLab_1.set_xy         (       getX, getY )
                                        BB.PDaLab_1.set_size       (       size.small )
                                        BB.PDaLab_1.set_textcolor  (    PDtxtCss )
                                        BB.PDaLab_1.set_text       ('Premium PD Array')
                                        BB.PDaLab_1.set_style(label.style_label_lower_left)
                                        cnt := 1                                        
                                        hh1 := getY
                                    1 =>
                                        if getY - getY2 > hh1
                                            BB.PDa_box2.set_lefttop    (getX,        getY )
                                            BB.PDaLine2.set_xy1        (getX,        getY )
                                            BB.PDa_box2.set_rightbottom(n   , getY - getY2)
                                            BB.PDaLine2.set_xy2        (n   , getY        )
                                            BB.PDa_box2.set_bgcolor    (       cSwingBl   )
                                            BB.PDaLab_2.set_xy         (       getX, getY )
                                            BB.PDaLab_2.set_size       (       size.small )
                                            BB.PDaLab_2.set_textcolor  (    PDtxtCss )
                                            BB.PDaLab_2.set_text       ('Premium PD Array')
                                            BB.PDaLab_2.set_style(label.style_label_lower_left)                                    
                                            cnt := 2
                            if cnt == 2
                                break                         

                        I  = green1prT - green1prB
                        E1 = green1prT + (I * R2a / R1a)
                        E2 = green1prT + (I * R2b / R1b)
                        E3 = green1prT + (I * R2c / R1c)

                        if iTP
                            if not BB.TP1_hit
                                BB.TP1_line.set_xy1(n, E1)  
                                BB.TP1_line.set_xy2(n + 20, E1)  
                            if not BB.TP2_hit
                                BB.TP2_line.set_xy1(n, E2)  
                                BB.TP2_line.set_xy2(n + 20, E2) 
                            if not BB.TP3_hit
                                BB.TP3_line.set_xy1(n, E3)  
                                BB.TP3_line.set_xy2(n + 20, E3) 

                        signals.set(BBplus, true)                        
                        alert('+BB', alert.freq_once_per_bar_close)
                        BB.aLabels.unshift(createLab('u', low, cBBplusB.notransp(), _arrowup, size.large))

                        break

            MSS.dir :=  1
            
        // MSS Bearish
        close < aZZ.y.get(iL) and aZZ.d.get(iL) == -1 and MSS.dir > -1 and per =>
            Ex   = aZZ.x.get(iL -1), Ey = aZZ.y.get(iL -1) 
            Dx   = aZZ.x.get(iL   ), Dy = aZZ.y.get(iL   ), DyMn = mn[n - Dx]
            Cx   = aZZ.x.get(iL +1), Cy = aZZ.y.get(iL +1) 
            Bx   = aZZ.x.get(iL +2), By = aZZ.y.get(iL +2), ByMn = mn[n - Bx] 
            Ax   = aZZ.x.get(iL +3), Ay = aZZ.y.get(iL +3), AyMx = mx[n - Ax]
            _y   = math.min(ByMn, DyMn)
            //_x   = _y == ByMn ? Bx : Dx
            mid  = AyMx - ((AyMx - _y) / 2) // 50% fib A- min(B, D)
            isOK = onlyWhenInPDarray ? Ay > Cy and Ay > Ey and Ey > mid : true
            //
            float red_1_prT = na
            float red_1_prB = na
            float    avg    = na
            if Ey > Cy and Cx != Dx and isOK 
                // latest LL to LL further -> search first red bar
                for i = n - Dx to n - Cx
                    if close[i] < open[i]
                        // reset previous swing box's
                        BB.PDa_box1.set_lefttop(na, na), BB.PDaLine1.set_xy1(na, na), BB.PDaLab_1.set_xy(na, na)
                        BB.PDa_box2.set_lefttop(na, na), BB.PDaLine2.set_xy1(na, na), BB.PDaLab_2.set_xy(na, na)

                        red_1_idx  = n - i
                        red_1_prT  := breakerCandleOnlyBody ? mx[i] : high[i]
                        red_1_prB  := breakerCandleOnlyBody ? mn[i] : low [i]
                        if breakerCandle_2Last 
                            if close[i +1] < open[i +1]
                                red_2_prT  = breakerCandleOnlyBody ? mx[i +1] : high[i +1]
                                red_2_prB  = breakerCandleOnlyBody ? mn[i +1] : low [i +1]
                                if red_2_prT > red_1_prT or red_2_prB < red_1_prB
                                    red_1_idx -= 1
                                red_1_prT := math.max(red_1_prT, red_2_prT)
                                red_1_prB := math.min(red_1_prB, red_2_prB)
                        
                        // Breaker Block -
                        avg := math.avg(red_1_prB, red_1_prT)
                        while BB.aLabels.size() > 0
                            BB.aLabels.pop().delete()
                        BB.PDa_boxA.delete(), BB.PDa_boxB.delete(), BB.dir := -1
                        BB.BB_boxA.set_left   (red_1_idx)
                        BB.BB_boxA.set_top    (red_1_prT)
                        BB.BB_boxA.set_right  (    n    )
                        BB.BB_boxA.set_bottom (red_1_prB)
                        BB.BB_boxA.set_bgcolor(cBB_minA )

                        BB.BB_boxB.set_left   (n)
                        BB.BB_boxB.set_top    (red_1_prT)
                        BB.BB_boxB.set_right  (    n + 8)
                        BB.BB_boxB.set_bottom (red_1_prB)
                        BB.BB_boxB.set_bgcolor(cBB_minB )
                        BB.BB_boxB.set_text('-BB')
                        BB.BB_boxB.set_text_color(cBB_minB.notransp())
                        BB.BB_boxB.set_text_valign(text.align_top)

                        BB.BB_line.set_xy1(n, avg), BB.BB_line.set_xy2(n , avg)

                        if showSPD
                            BB.line_1.set_xy1(Cx, Cy), BB.line_1.set_xy2(n , Cy), BB.Broken_1 := false
                            BB.line_2.set_xy1(Ex, Ey), BB.line_2.set_xy2(n , Ey), BB.Broken_2 := false
                            BB.HL.set_xy(Ex, Ey), BB.HL.set_style(label.style_label_down), BB.HL.set_text('HH'), BB.HL.set_textcolor(PDtxtCss)

                        BB.TP1_hit    := false     
                        BB.TP2_hit    := false                              
                        BB.TP3_hit    := false    
                        BB.Broken     := false
                        BB.Mitigated  := false   
                        BB.scalp      := false
                        BB.PDbroken1  := false
                        BB.PDbroken2  := false

                        if onlyWhenInPDarray and showPDarray
                            BB.PDa_boxA := box.new(Ax, AyMx, Ex +1, mid, bgcolor=color.rgb(248, 153, 132, 90), border_color=color(na)
                             , text = 'Premium PD Array', text_size = size.small, text_color = color.rgb(248, 153, 132, 25)
                             , text_halign = text.align_right, text_valign = text.align_center, text_font_family = font.family_monospace) // , text_wrap= text.wrap_auto
                            BB.PDa_boxB := box.new(Ax, mid , Ex +1,  _y, bgcolor=color.rgb(132, 248, 171, 90), border_color=color(na))

                        // Previous swings
                        cnt = 0, ll1 = low
                        for c = 0 to sz -2
                            getX = aZZ.x.get(c)
                            getY = aZZ.y.get(c)
                            if getY < ll1 and aZZ.d.get(c) == -1 and showSPD
                                getY2 = (mx[n - getX] - low[n - getX]) / 4
                                switch cnt 
                                    0 =>
                                        BB.PDa_box1.set_lefttop    (getX, getY + getY2)
                                        BB.PDaLine1.set_xy1        (getX,        getY )
                                        BB.PDa_box1.set_rightbottom(       n   , getY )
                                        BB.PDaLine1.set_xy2        (       n   , getY )
                                        BB.PDa_box1.set_bgcolor    (       cSwingBr   )
                                        BB.PDaLab_1.set_xy         (       getX, getY )
                                        BB.PDaLab_1.set_size       (       size.small )
                                        BB.PDaLab_1.set_textcolor  (    PDtxtCss )
                                        BB.PDaLab_1.set_text      ('Discount PD Array')
                                        BB.PDaLab_1.set_style(label.style_label_upper_left)

                                        cnt := 1
                                        ll1 := getY
                                    1 => 
                                        if getY + getY2 < ll1
                                            BB.PDa_box2.set_lefttop    (getX, getY + getY2)
                                            BB.PDaLine2.set_xy1        (getX,        getY )
                                            BB.PDa_box2.set_rightbottom(       n   , getY )
                                            BB.PDaLine2.set_xy2        (       n   , getY )
                                            BB.PDa_box2.set_bgcolor    (       cSwingBr   )
                                            BB.PDaLab_2.set_xy         (       getX, getY )
                                            BB.PDaLab_2.set_size       (       size.small )
                                            BB.PDaLab_2.set_textcolor  (    PDtxtCss )
                                            BB.PDaLab_2.set_text      ('Discount PD Array')
                                            BB.PDaLab_2.set_style(label.style_label_upper_left)                                       
                                            cnt := 2
                            if cnt == 2
                                break  

                        I  = red_1_prT - red_1_prB
                        E1 = red_1_prB - (I * R2a / R1a)
                        E2 = red_1_prB - (I * R2b / R1b)
                        E3 = red_1_prB - (I * R2c / R1c)

                        if iTP
                            if not BB.TP1_hit
                                BB.TP1_line.set_xy1(n, E1)  
                                BB.TP1_line.set_xy2(n + 20, E1)  
                            if not BB.TP2_hit
                                BB.TP2_line.set_xy1(n, E2)  
                                BB.TP2_line.set_xy2(n + 20, E2) 
                            if not BB.TP3_hit
                                BB.TP3_line.set_xy1(n, E3)  
                                BB.TP3_line.set_xy2(n + 20, E3) 

                        signals.set(BB_min, true)                        
                        alert('-BB', alert.freq_once_per_bar_close)
                        BB.aLabels.unshift(createLab('d', high, cBB_minB.notransp(), _arrowdn, size.large))
                        
                        break       
                
            MSS.dir := -1 

//-----------------------------------------------------------------------------}
//Calculations
//-----------------------------------------------------------------------------{
draw(len, tpCss)  


lft = BB.BB_boxB.get_left  ()
top = BB.BB_boxB.get_top   ()
btm = BB.BB_boxB.get_bottom() 
avg = BB.BB_line.get_y2    ()
l_1 = BB.line_1.get_y2     ()
l_2 = BB.line_2.get_y2     ()
TP1 = BB.TP1_line.get_y2   ()
TP2 = BB.TP2_line.get_y2   ()
TP3 = BB.TP3_line.get_y2   ()

switch BB.dir
    1  => 
        if not BB.Mitigated
            if close < btm
                BB.Mitigated := true 
                signals.set(BB_endBl, true)     
                alert('+BB Mitigated', alert.freq_once_per_bar_close)

                BB.aLabels.unshift(createLab('u', low, color.yellow, _c))
                
                BB.BB_boxB.set_right(n)
                BB.BB_line.set_x2   (n)
            else
                BB.BB_boxB.set_right(n + 8)
                BB.BB_line.set_x2   (n + 8)
                
            BB.TP1_line.set_x2   (n)
            BB.TP2_line.set_x2   (n)
            BB.TP3_line.set_x2   (n)

            if n > BB.BB_boxB.get_left()
                if not BB.Broken
                    if BB.scalp
                        if not BB.TP1_hit and open < TP1 and high > TP1
                            BB.TP1_hit := true
                            signals.set(tpUP1, true)     
                            alert('TP UP 1', alert.freq_once_per_bar)
                            BB.aLabels.unshift(createLab('c', TP1, #ff00dd, _c))
                        if not BB.TP2_hit and open < TP2 and high > TP2
                            BB.TP2_hit := true                                 
                            signals.set(tpUP2, true)     
                            alert('TP UP 2', alert.freq_once_per_bar)
                            BB.aLabels.unshift(createLab('c', TP2, #ff00dd, _c))
                        if not BB.TP3_hit and open < TP3 and high > TP3
                            BB.TP3_hit := true                        
                            signals.set(tpUP3, true)     
                            alert('TP UP 3', alert.freq_once_per_bar)
                            BB.aLabels.unshift(createLab('c', TP3, #ff00dd, _c))
                    switch
                        open > avg and open < top and close > top => 
                            BB.TP1_hit := false
                            BB.TP2_hit := false
                            BB.TP3_hit := false
                            BB.scalp   := true
                            signals.set(signUP, true)                        
                            alert('signal UP', alert.freq_once_per_bar_close)
                            BB.aLabels.unshift(createLab('u', low, color.lime, _arrowup, size.normal))
                        close < avg and close > btm => 
                            BB.Broken := true
                            BB.scalp  := false
                            signals.set(cnclUP, true)                        
                            alert('cancel UP', alert.freq_once_per_bar_close)
                            BB.aLabels.unshift(createLab('u', low, color.orange, _x))
                else
                    // reset
                    if not tillFirstBreak and close > top 
                        BB.Broken := false  
                        BB.scalp := true 
                        signals.set(BBplus, true)                        
                        alert('+BB (R)', alert.freq_once_per_bar_close)
                        BB.aLabels.unshift(createLab('u', low, color.blue, 'R', size.normal)) 

        if not BB.Broken_1
            BB.line_1.set_x2(n)
            if close < l_1
                BB.Broken_1 := true
                signals.set(LL1break, true)                        
                alert('LL 1 break', alert.freq_once_per_bar_close)
                if showBreaks
                    BB.aLabels.unshift(createLab('c', low, #c00000, _c))
        if not BB.Broken_2 
            BB.line_2.set_x2(n)
            if close < l_2
                BB.Broken_2 := true                     
                signals.set(LL2break, true)                        
                alert('LL 2 break', alert.freq_once_per_bar_close)
                if showBreaks
                    BB.aLabels.unshift(createLab('c', low, #c00000, _c))

        if not BB.PDbroken1
            BB.PDa_box1.set_right(n)            
            BB.PDaLine1.set_x2   (n)
            if close > BB.PDa_box1.get_top() and n > BB.PDa_box1.get_left()
                BB.PDbroken1 := true             
                signals.set(SW1breakUP, true)                       
                alert('Swing UP 1 break', alert.freq_once_per_bar_close)
                if showBreaks
                    BB.aLabels.unshift(createLab('c', high, #c00000, _c))
        if not BB.PDbroken2
            BB.PDa_box2.set_right(n)            
            BB.PDaLine2.set_x2   (n)
            if close > BB.PDa_box2.get_top() and n > BB.PDa_box2.get_left()
                BB.PDbroken2 := true                 
                signals.set(SW2breakUP, true)                        
                alert('Swing UP 2 break', alert.freq_once_per_bar_close)
                if showBreaks
                    BB.aLabels.unshift(createLab('c', high, #c00000, _c))

    -1 =>
        if not BB.Mitigated
            if close > top
                BB.Mitigated := true 
                signals.set(BB_endBr, true)     
                alert('-BB Mitigated', alert.freq_once_per_bar_close)
                if showBreaks
                    BB.aLabels.unshift(createLab('d', high, cBB_minB.notransp(), _c))
                BB.BB_boxB.set_right(n)
                BB.BB_line.set_x2   (n)
            else
                BB.BB_boxB.set_right(n + 8)
                BB.BB_line.set_x2   (n + 8)

            BB.TP1_line.set_x2   (n)
            BB.TP2_line.set_x2   (n)
            BB.TP3_line.set_x2   (n)

            if n > BB.BB_boxB.get_left()
                if not BB.Broken
                    if BB.scalp
                        if not BB.TP1_hit and open > TP1 and low < TP1
                            BB.TP1_hit := true                       
                            signals.set(tpDN1, true)                             
                            alert('TP DN 1', alert.freq_once_per_bar)
                            BB.aLabels.unshift(createLab('c', TP1, #ff00dd, _c))
                        if not BB.TP2_hit and open > TP2 and low < TP2
                            BB.TP2_hit := true                                 
                            signals.set(tpDN2, true)                             
                            alert('TP DN 2', alert.freq_once_per_bar)               
                            BB.aLabels.unshift(createLab('c', TP2, #ff00dd, _c))
                        if not BB.TP3_hit and open > TP3 and low < TP3
                            BB.TP3_hit := true                                    
                            signals.set(tpDN3, true)                             
                            alert('TP DN 3', alert.freq_once_per_bar)       
                            BB.aLabels.unshift(createLab('c', TP3, #ff00dd, _c))
                    switch
                        open < avg and open > btm and close < btm => 
                            BB.TP1_hit := false
                            BB.TP2_hit := false
                            BB.TP3_hit := false
                            BB.scalp   := true
                            signals.set(signDN, true)
                            alert('signal DN', alert.freq_once_per_bar_close)
                            BB.aLabels.unshift(createLab('d', high, color.orange, _arrowdn, size.normal))
                        close > avg and close < top => 
                            BB.Broken := true
                            BB.scalp  := false
                            signals.set(cnclDN, true)
                            alert('cancel DN', alert.freq_once_per_bar_close)
                            BB.aLabels.unshift(createLab('d', high, color.red   , _x))
                else
                    // reset
                    if not tillFirstBreak and close < btm 
                        BB.Broken := false 
                        BB.scalp  := true 
                        signals.set(BB_min, true)                        
                        alert('-BB (R)', alert.freq_once_per_bar_close)                        
                        BB.aLabels.unshift(createLab('d', high, color.blue, 'R', size.normal))

        if not BB.Broken_1             
            BB.line_1.set_x2(n)
            if close > l_1                 
                BB.Broken_1 := true
                signals.set(HH1break, true)                        
                alert('HH 1 break', alert.freq_once_per_bar_close)
                if showBreaks
                    BB.aLabels.unshift(createLab('c', high, #c00000, _c))
        if not BB.Broken_2             
            BB.line_2.set_x2(n)
            if close > l_2
                BB.Broken_2 := true
                signals.set(HH2break, true)                        
                alert('HH 2 break', alert.freq_once_per_bar_close)
                if showBreaks
                    BB.aLabels.unshift(createLab('c', high, #c00000, _c))

        if not BB.PDbroken1
            BB.PDa_box1.set_right(n)
            BB.PDaLine1.set_x2   (n)
            if close < BB.PDa_box1.get_bottom() and n > BB.PDa_box1.get_left()
                BB.PDbroken1 := true
                signals.set(SW1breakDN, true)                        
                alert('Swing DN 1 break', alert.freq_once_per_bar_close)
                if showBreaks
                    BB.aLabels.unshift(createLab('c', low, #c00000, _c))
        if not BB.PDbroken2
            BB.PDa_box2.set_right(n)            
            BB.PDaLine2.set_x2   (n)
            if close < BB.PDa_box2.get_bottom() and n > BB.PDa_box2.get_left()
                BB.PDbroken2 := true
                signals.set(SW2breakDN, true)                        
                alert('Swing DN 2 break', alert.freq_once_per_bar_close)
                if showBreaks
                    BB.aLabels.unshift(createLab('c', low, #c00000, _c))
  
//-----------------------------------------------------------------------------}
//Alerts
//-----------------------------------------------------------------------------{
alertcondition(signals.get(BBplus    ), ' 1. +BB'             , '1. +BB'             )
alertcondition(signals.get(signUP    ), ' 2. signal UP'       , '2. signal UP'       )
alertcondition(signals.get(tpUP1     ), ' 3. TP UP 1'         , '3. TP UP 1'         )
alertcondition(signals.get(tpUP2     ), ' 3. TP UP 2'         , '3. TP UP 2'         )
alertcondition(signals.get(tpUP3     ), ' 3. TP UP 3'         , '3. TP UP 3'         )
alertcondition(signals.get(cnclUP    ), ' 4. cancel UP'       , '4. cancel UP'       )
alertcondition(signals.get(BB_endBl  ), ' 5. +BB Mitigated'   , '5. +BB Mitigated'   )
alertcondition(signals.get(LL1break  ), ' 6. LL 1 Break'      , '6. LL 1 Break'      )
alertcondition(signals.get(LL2break  ), ' 6. LL 2 Break'      , '6. LL 2 Break'      )
alertcondition(signals.get(SW1breakUP), ' 7. Swing UP 1 Break', '7. Swing UP 1 Break')
alertcondition(signals.get(SW2breakUP), ' 7. Swing UP 2 Break', '7. Swing UP 2 Break')

alertcondition(signals.get(BB_min    ),  '1. -BB'             , '1. -BB'             )
alertcondition(signals.get(signDN    ),  '2. signal DN'       , '2. signal DN'       )
alertcondition(signals.get(tpDN1     ),  '3. TP DN 1'         , '3. TP DN 1'         )
alertcondition(signals.get(tpDN2     ),  '3. TP DN 2'         , '3. TP DN 2'         )
alertcondition(signals.get(tpDN3     ),  '3. TP DN 3'         , '3. TP DN 3'         )
alertcondition(signals.get(cnclDN    ),  '4. cancel DN'       , '4. cancel DN'       )
alertcondition(signals.get(BB_endBr  ),  '5. -BB Mitigated'   , '5. -BB Mitigated'   )
alertcondition(signals.get(HH1break  ),  '6. HH 1 Break'      , '6. HH 1 Break'      )
alertcondition(signals.get(HH2break  ),  '6. HH 2 Break'      , '6. HH 2 Break'      )
alertcondition(signals.get(SW1breakDN),  '7. Swing DN 1 Break', '7. Swing DN 1 Break')
alertcondition(signals.get(SW2breakDN),  '7. Swing DN 2 Break', '7. Swing DN 2 Break')

//-----------------------------------------------------------------------------}

2.Imbalance Detector [LuxAlgo]
// This work is licensed under a Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0) https://creativecommons.org/licenses/by-nc-sa/4.0/
// © LuxAlgo

//@version=5
indicator("Imbalance Detector [LuxAlgo]"
  , overlay = true
  , max_boxes_count = 500
  , max_labels_count = 500
  , max_lines_count = 500)
//------------------------------------------------------------------------------
//Settings
//-----------------------------------------------------------------------------{
//Fair Value Gaps
show_fvg = input(true, 'Fair Value Gaps (FVG)'
  , inline = 'fvg_css'
  , group = 'Fair Value Gaps')

bull_fvg_css = input.color(#2157f3, ''
  , inline = 'fvg_css'
  , group = 'Fair Value Gaps')

bear_fvg_css = input.color(#ff1100, ''
  , inline = 'fvg_css'
  , group = 'Fair Value Gaps')

fvg_usewidth = input(false, 'Min Width'
  , inline = 'fvg_width'
  , group = 'Fair Value Gaps')

fvg_gapwidth = input.float(0, ''
  , inline = 'fvg_width'
  , group = 'Fair Value Gaps')

fvg_method = input.string('Points', ''
  , options = ['Points', '%', 'ATR']
  , inline = 'fvg_width'
  , group = 'Fair Value Gaps')

fvg_extend = input.int(0, 'Extend FVG'
  , minval = 0
  , group = 'Fair Value Gaps')

//Opening Gaps
show_og = input(true, 'Show Opening Gaps (OG)'
  , inline = 'og_css'
  , group = 'Opening Gaps')

bull_og_css = input.color(#2157f3, ''
  , inline = 'og_css'
  , group = 'Opening Gaps')

bear_og_css = input.color(#ff1100, ''
  , inline = 'og_css'
  , group = 'Opening Gaps')

og_usewidth = input(false, 'Min Width'
  , inline = 'og_width'
  , group = 'Opening Gaps')

og_gapwidth = input.float(0, ''
  , inline = 'og_width'
  , group = 'Opening Gaps')

og_method = input.string('Points', ''
  , options = ['Points', '%', 'ATR']
  , inline = 'og_width'
  , group = 'Opening Gaps')

og_extend = input.int(0, 'Extend OG'
  , minval = 0
  , group = 'Opening Gaps')

//Volume Imbalance
show_vi = input(true, 'Show Volume Imbalances (VI)'
  , inline = 'vi_css'
  , group = 'Volume Imbalance')

bull_vi_css = input.color(#2157f3, ''
  , inline = 'vi_css'
  , group = 'Volume Imbalance')

bear_vi_css = input.color(#ff1100, ''
  , inline = 'vi_css'
  , group = 'Volume Imbalance')

vi_usewidth = input(false, 'Min Width'
  , inline = 'vi_width'
  , group = 'Volume Imbalance')

vi_gapwidth = input.float(0, ''
  , inline = 'vi_width'
  , group = 'Volume Imbalance')

vi_method = input.string('Points', ''
  , options = ['Points', '%', 'ATR']
  , inline = 'vi_width'
  , group = 'Volume Imbalance')

vi_extend = input.int(5, 'Extend VI'
  , minval = 0
  , group = 'Volume Imbalance')

//Dashboard
show_dash = input(false, 'Show Dashboard'
  , group   = 'Dashboard')

dash_loc = input.string('Bottom Right', 'Dashboard Location'
  , options = ['Top Right', 'Bottom Right', 'Bottom Left']
  , group   = 'Dashboard')
  
text_size = input.string('Tiny', 'Dashboard Size'
  , options = ['Tiny', 'Small', 'Normal']
  , group   = 'Dashboard')

//-----------------------------------------------------------------------------}
//Functions
//-----------------------------------------------------------------------------{
n = bar_index
atr = ta.atr(200)

//Detect imbalance and return count over time
imbalance_detection(show, usewidth, method, width, top, btm, condition)=>
    var is_width = true
    var count = 0

    if usewidth
        dist = top - btm

        is_width := switch method
            'Points' => dist > width
            '%' => dist / btm * 100 > width
            'ATR' => dist > atr * width

    is_true = show and condition and is_width
    count += is_true ? 1 : 0
    
    [is_true, count]

//Detect if bullish imbalance is filled and return count over time
bull_filled(condition, btm)=>
    var btms = array.new_float(0)
    var count = 0

    if condition
        array.unshift(btms, btm)
    
    size = array.size(btms)

    for i = (size > 0 ? size-1 : na) to 0
        value = array.get(btms, i)

        if low < value
            array.remove(btms, i)      
            count += 1 

    count

//Detect if bearish imbalance is filled and return count over time
bear_filled(condition, top)=>
    var tops = array.new_float(0)
    var count = 0
    
    if condition
        array.unshift(tops, top)
    
    size = array.size(tops)
    
    for i = (size > 0 ? size-1 : na) to 0
        value = array.get(tops, i)

        if high > value
            array.remove(tops, i)      
            count += 1 

    count

//Set table data cells
set_cells(tb, column, bull_filled, bull_count, bear_filled, bear_count, bull_css, bear_css, table_size)=>
    table.cell(tb, column, 1, str.tostring(bull_count)
      , text_color = bull_css
      , text_size = table_size
      , text_halign = text.align_left)

    table.cell(tb, column, 2, str.format('{0,number,percent}', bull_filled / bull_count)
      , text_color = bull_css
      , text_size = table_size
      , text_halign = text.align_left)

    table.cell(tb, column, 3, str.tostring(bear_count)
      , text_color = bear_css
      , text_size = table_size
      , text_halign = text.align_left)

    table.cell(tb, column, 4, str.format('{0,number,percent}', bear_filled / bear_count)
      , text_color = bear_css
      , text_size = table_size
      , text_halign = text.align_left)

//-----------------------------------------------------------------------------}
//Volume Imbalances
//-----------------------------------------------------------------------------{
//Bullish
bull_gap_top = math.min(close, open)
bull_gap_btm = math.max(close[1], open[1])

[bull_vi, bull_vi_count] = imbalance_detection(
  show_vi
  , vi_usewidth
  , vi_method
  , vi_gapwidth
  , bull_gap_top
  , bull_gap_btm
  , open > close[1] and high[1] > low and close > close[1] and open > open[1] and high[1] < bull_gap_top)

bull_vi_filled = bull_filled(bull_vi[1], bull_gap_btm[1])

//Bearish
bear_gap_top = math.min(close[1], open[1])
bear_gap_btm = math.max(close, open)

[bear_vi, bear_vi_count] = imbalance_detection(
  show_vi
  , vi_usewidth
  , vi_method
  , vi_gapwidth
  , bear_gap_top
  , bear_gap_btm
  , open < close[1] and low[1] < high and close < close[1] and open < open[1] and low[1] > bear_gap_btm)

bear_vi_filled = bear_filled(bear_vi[1], bear_gap_top[1])

if bull_vi
    box.new(n-1, bull_gap_top, n + vi_extend, bull_gap_btm, border_color = bull_vi_css, bgcolor = na, border_style = line.style_dotted)
    
if bear_vi
    box.new(n-1, bear_gap_top, n + vi_extend, bear_gap_btm, border_color = bear_vi_css, bgcolor = na, border_style = line.style_dotted)

//-----------------------------------------------------------------------------}
//Opening Gaps
//-----------------------------------------------------------------------------{
//Bullish
[bull_og, bull_og_count] = imbalance_detection(
  show_og
  , og_usewidth
  , og_method
  , og_gapwidth
  , bull_gap_top
  , bull_gap_btm
  , low > high[1])

bull_og_filled = bull_filled(bull_og, bull_gap_btm)

//Bullish
[bear_og, bear_og_count] = imbalance_detection(
  show_og
  , og_usewidth
  , og_method
  , og_gapwidth
  , bear_gap_top
  , bear_gap_btm
  , high < low[1])

bear_og_filled = bear_filled(bear_og, bear_gap_top)

if bull_og
    box.new(n-1, bull_gap_top, n + og_extend, bull_gap_btm, border_color = na, bgcolor = color.new(bull_og_css, 50), text = 'OG', text_color = chart.fg_color)

if bear_og
    box.new(n-1, bear_gap_top, n + og_extend, bear_gap_btm, border_color = na, bgcolor = color.new(bear_og_css, 50), text = 'OG', text_color = chart.fg_color)

//-----------------------------------------------------------------------------}
//Fair Value Gaps
//-----------------------------------------------------------------------------{
//Bullish
[bull_fvg, bull_fvg_count] = imbalance_detection(
  show_fvg
  , fvg_usewidth
  , fvg_method
  , fvg_gapwidth
  , low
  , high[2]
  , low > high[2] and close[1] > high[2] and not (bull_og or bull_og[1]))

bull_fvg_filled = bull_filled(bull_fvg, high[2])

//Bearish
[bear_fvg, bear_fvg_count] = imbalance_detection(
  show_fvg
  , fvg_usewidth
  , fvg_method
  , fvg_gapwidth
  , low[2]
  , high
  , high < low[2] and close[1] < low[2] and not (bear_og or bear_og[1]))

bear_fvg_filled = bear_filled(bear_fvg, low[2])

if bull_fvg
    avg = math.avg(low, high[2])
    
    box.new(n-2, low, n + fvg_extend, high[2], border_color = na, bgcolor = color.new(bull_fvg_css, 80))
    line.new(n-2, avg, n + fvg_extend, avg, color = bull_fvg_css)
    
if bear_fvg
    avg = math.avg(low[2], high)

    box.new(n-2, low[2], n + fvg_extend, high, border_color = na, bgcolor = color.new(bear_fvg_css, 80))
    line.new(n-2, avg, n + fvg_extend, avg, color = bear_fvg_css)

//-----------------------------------------------------------------------------}
//Dashboard
//-----------------------------------------------------------------------------{
var table_position = dash_loc == 'Bottom Left' ? position.bottom_left 
  : dash_loc == 'Top Right' ? position.top_right 
  : position.bottom_right

var table_size = text_size == 'Tiny' ? size.tiny 
  : text_size == 'Small' ? size.small 
  : size.normal

var table tb = na

//Set table legends
if barstate.isfirst and show_dash
    tb := table.new(table_position, 5, 7
      , bgcolor = #1e222d
      , border_color = #373a46
      , border_width = 1
      , frame_color = #373a46
      , frame_width = 1)
    
    if show_fvg
        table.cell(tb, 2, 0, '〓 FVG', text_color = color.white, text_size = table_size)

    if show_og
        table.cell(tb, 3, 0, '◼ OG', text_color = color.white, text_size = table_size)

    if show_vi
        table.cell(tb, 4, 0, '⬚ VI', text_color = color.white, text_size = table_size)
    
    //Set vertical legends
    table.merge_cells(tb, 0, 0, 1, 0)

    table.cell(tb, 0, 1, 'Bullish', text_color = #089981, text_size = table_size)
    
    table.cell(tb, 1, 1, 'Frequency', text_color = #089981, text_size = table_size, text_halign = text.align_left)
    
    table.cell(tb, 1, 2, 'Filled', text_color = #089981, text_size = table_size, text_halign = text.align_left)
    
    table.merge_cells(tb, 0, 1, 0, 2)
    
    table.cell(tb, 0, 3, 'Bearish', text_color = #f23645, text_size = table_size)
    
    table.cell(tb, 1, 3, 'Frequency', text_color = #f23645, text_size = table_size, text_halign = text.align_left)
    
    table.cell(tb, 1, 4, 'Filled', text_color = #f23645, text_size = table_size, text_halign = text.align_left)

    table.merge_cells(tb, 0, 3, 0, 4)

//Set dashboard data
if barstate.islast and show_dash
    if show_fvg
        set_cells(tb, 2, bull_fvg_filled, bull_fvg_count, bear_fvg_filled, bear_fvg_count, bull_fvg_css, bear_fvg_css, table_size)

    if show_og
        set_cells(tb, 3, bull_og_filled, bull_og_count, bear_og_filled, bear_og_count, bull_og_css, bear_og_css, table_size)
        
    if show_vi
        set_cells(tb, 4, bull_vi_filled, bull_vi_count, bear_vi_filled, bear_vi_count, bull_vi_css, bear_vi_css, table_size)

//-----------------------------------------------------------------------------}
//Alerts
//-----------------------------------------------------------------------------{
//FVG
alertcondition(bull_fvg, 'Bullish FVG', 'Bullish FVG detected')
alertcondition(bear_fvg, 'Bearish FVG', 'Bearish FVG detected')

//OG
alertcondition(bull_og, 'Bullish OG', 'Bullish OG detected')
alertcondition(bear_og, 'Bearish OG', 'Bearish OG detected')

//VI
alertcondition(bull_vi, 'Bullish VI', 'Bullish VI detected')
alertcondition(bear_vi, 'Bearish VI', 'Bearish VI detected')

//-----------------------------------------------------------------------------}

3.Smart Money Concepts (SMC) [LuxAlgo]
// This work is licensed under a Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0) https://creativecommons.org/licenses/by-nc-sa/4.0/
// © LuxAlgo

//@version=5
indicator('Smart Money Concepts [LuxAlgo]', 'LuxAlgo - Smart Money Concepts', overlay = true, max_labels_count = 500, max_lines_count = 500, max_boxes_count = 500)
//---------------------------------------------------------------------------------------------------------------------}
//CONSTANTS & STRINGS & INPUTS
//---------------------------------------------------------------------------------------------------------------------{
BULLISH_LEG                     = 1
BEARISH_LEG                     = 0

BULLISH                         = +1
BEARISH                         = -1

GREEN                           = #089981
RED                             = #F23645
BLUE                            = #2157f3
GRAY                            = #878b94
MONO_BULLISH                    = #b2b5be
MONO_BEARISH                    = #5d606b

HISTORICAL                      = 'Historical'
PRESENT                         = 'Present'

COLORED                         = 'Colored'
MONOCHROME                      = 'Monochrome'

ALL                             = 'All'
BOS                             = 'BOS'
CHOCH                           = 'CHoCH'

TINY                            = size.tiny
SMALL                           = size.small
NORMAL                          = size.normal

ATR                             = 'Atr'
RANGE                           = 'Cumulative Mean Range'

CLOSE                           = 'Close'
HIGHLOW                         = 'High/Low'

SOLID                           = '⎯⎯⎯'
DASHED                          = '----'
DOTTED                          = '····'

SMART_GROUP                     = 'Smart Money Concepts'
INTERNAL_GROUP                  = 'Real Time Internal Structure'
SWING_GROUP                     = 'Real Time Swing Structure'
BLOCKS_GROUP                    = 'Order Blocks'
EQUAL_GROUP                     = 'EQH/EQL'
GAPS_GROUP                      = 'Fair Value Gaps'
LEVELS_GROUP                    = 'Highs & Lows MTF'
ZONES_GROUP                     = 'Premium & Discount Zones'

modeTooltip                     = 'Allows to display historical Structure or only the recent ones'
styleTooltip                    = 'Indicator color theme'
showTrendTooltip                = 'Display additional candles with a color reflecting the current trend detected by structure'
showInternalsTooltip            = 'Display internal market structure'
internalFilterConfluenceTooltip = 'Filter non significant internal structure breakouts'
showStructureTooltip            = 'Display swing market Structure'
showSwingsTooltip               = 'Display swing point as labels on the chart'
showHighLowSwingsTooltip        = 'Highlight most recent strong and weak high/low points on the chart'
showInternalOrderBlocksTooltip  = 'Display internal order blocks on the chart\n\nNumber of internal order blocks to display on the chart'
showSwingOrderBlocksTooltip     = 'Display swing order blocks on the chart\n\nNumber of internal swing blocks to display on the chart'
orderBlockFilterTooltip         = 'Method used to filter out volatile order blocks \n\nIt is recommended to use the cumulative mean range method when a low amount of data is available'
orderBlockMitigationTooltip     = 'Select what values to use for order block mitigation'
showEqualHighsLowsTooltip       = 'Display equal highs and equal lows on the chart'
equalHighsLowsLengthTooltip     = 'Number of bars used to confirm equal highs and equal lows'
equalHighsLowsThresholdTooltip  = 'Sensitivity threshold in a range (0, 1) used for the detection of equal highs & lows\n\nLower values will return fewer but more pertinent results'
showFairValueGapsTooltip        = 'Display fair values gaps on the chart'
fairValueGapsThresholdTooltip   = 'Filter out non significant fair value gaps'
fairValueGapsTimeframeTooltip   = 'Fair value gaps timeframe'
fairValueGapsExtendTooltip      = 'Determine how many bars to extend the Fair Value Gap boxes on chart'
showPremiumDiscountZonesTooltip = 'Display premium, discount, and equilibrium zones on chart'

modeInput                       = input.string( HISTORICAL, 'Mode',                     group = SMART_GROUP,    tooltip = modeTooltip, options = [HISTORICAL, PRESENT])
styleInput                      = input.string( COLORED,    'Style',                    group = SMART_GROUP,    tooltip = styleTooltip,options = [COLORED, MONOCHROME])
showTrendInput                  = input(        false,      'Color Candles',            group = SMART_GROUP,    tooltip = showTrendTooltip)

showInternalsInput              = input(        true,       'Show Internal Structure',  group = INTERNAL_GROUP, tooltip = showInternalsTooltip)
showInternalBullInput           = input.string( ALL,        'Bullish Structure',        group = INTERNAL_GROUP, inline = 'ibull', options = [ALL,BOS,CHOCH])
internalBullColorInput          = input(        GREEN,      '',                         group = INTERNAL_GROUP, inline = 'ibull')
showInternalBearInput           = input.string( ALL,        'Bearish Structure' ,       group = INTERNAL_GROUP, inline = 'ibear', options = [ALL,BOS,CHOCH])
internalBearColorInput          = input(        RED,        '',                         group = INTERNAL_GROUP, inline = 'ibear')
internalFilterConfluenceInput   = input(        false,      'Confluence Filter',        group = INTERNAL_GROUP, tooltip = internalFilterConfluenceTooltip)
internalStructureSize           = input.string( TINY,       'Internal Label Size',      group = INTERNAL_GROUP, options = [TINY,SMALL,NORMAL])

showStructureInput              = input(        true,       'Show Swing Structure',     group = SWING_GROUP,    tooltip = showStructureTooltip)
showSwingBullInput              = input.string( ALL,        'Bullish Structure',        group = SWING_GROUP,    inline = 'bull',    options = [ALL,BOS,CHOCH])
swingBullColorInput             = input(        GREEN,      '',                         group = SWING_GROUP,    inline = 'bull')
showSwingBearInput              = input.string( ALL,        'Bearish Structure',        group = SWING_GROUP,    inline = 'bear',    options = [ALL,BOS,CHOCH])
swingBearColorInput             = input(        RED,        '',                         group = SWING_GROUP,    inline = 'bear')
swingStructureSize              = input.string( SMALL,      'Swing Label Size',         group = SWING_GROUP,    options = [TINY,SMALL,NORMAL])
showSwingsInput                 = input(        false,      'Show Swings Points',       group = SWING_GROUP,    tooltip = showSwingsTooltip,inline = 'swings')
swingsLengthInput               = input.int(    50,         '',                         group = SWING_GROUP,    minval = 10,                inline = 'swings')
showHighLowSwingsInput          = input(        true,       'Show Strong/Weak High/Low',group = SWING_GROUP,    tooltip = showHighLowSwingsTooltip)

showInternalOrderBlocksInput    = input(        true,       'Internal Order Blocks' ,   group = BLOCKS_GROUP,   tooltip = showInternalOrderBlocksTooltip,   inline = 'iob')
internalOrderBlocksSizeInput    = input.int(    5,          '',                         group = BLOCKS_GROUP,   minval = 1, maxval = 20,                    inline = 'iob')
showSwingOrderBlocksInput       = input(        false,      'Swing Order Blocks',       group = BLOCKS_GROUP,   tooltip = showSwingOrderBlocksTooltip,      inline = 'ob')
swingOrderBlocksSizeInput       = input.int(    5,          '',                         group = BLOCKS_GROUP,   minval = 1, maxval = 20,                    inline = 'ob') 
orderBlockFilterInput           = input.string( 'Atr',      'Order Block Filter',       group = BLOCKS_GROUP,   tooltip = orderBlockFilterTooltip,          options = [ATR, RANGE])
orderBlockMitigationInput       = input.string( HIGHLOW,    'Order Block Mitigation',   group = BLOCKS_GROUP,   tooltip = orderBlockMitigationTooltip,      options = [CLOSE,HIGHLOW])
internalBullishOrderBlockColor  = input.color(color.new(#3179f5, 80), 'Internal Bullish OB',    group = BLOCKS_GROUP)
internalBearishOrderBlockColor  = input.color(color.new(#f77c80, 80), 'Internal Bearish OB',    group = BLOCKS_GROUP)
swingBullishOrderBlockColor     = input.color(color.new(#1848cc, 80), 'Bullish OB',             group = BLOCKS_GROUP)
swingBearishOrderBlockColor     = input.color(color.new(#b22833, 80), 'Bearish OB',             group = BLOCKS_GROUP)

showEqualHighsLowsInput         = input(        true,       'Equal High/Low',           group = EQUAL_GROUP,    tooltip = showEqualHighsLowsTooltip)
equalHighsLowsLengthInput       = input.int(    3,          'Bars Confirmation',        group = EQUAL_GROUP,    tooltip = equalHighsLowsLengthTooltip,      minval = 1)
equalHighsLowsThresholdInput    = input.float(  0.1,        'Threshold',                group = EQUAL_GROUP,    tooltip = equalHighsLowsThresholdTooltip,   minval = 0, maxval = 0.5, step = 0.1)
equalHighsLowsSizeInput         = input.string( TINY,       'Label Size',               group = EQUAL_GROUP,    options = [TINY,SMALL,NORMAL])

showFairValueGapsInput          = input(        false,      'Fair Value Gaps',          group = GAPS_GROUP,     tooltip = showFairValueGapsTooltip)
fairValueGapsThresholdInput     = input(        true,       'Auto Threshold',           group = GAPS_GROUP,     tooltip = fairValueGapsThresholdTooltip)
fairValueGapsTimeframeInput     = input.timeframe('',       'Timeframe',                group = GAPS_GROUP,     tooltip = fairValueGapsTimeframeTooltip)
fairValueGapsBullColorInput     = input.color(color.new(#00ff68, 70), 'Bullish FVG' , group = GAPS_GROUP)
fairValueGapsBearColorInput     = input.color(color.new(#ff0008, 70), 'Bearish FVG' , group = GAPS_GROUP)
fairValueGapsExtendInput        = input.int(    1,          'Extend FVG',               group = GAPS_GROUP,     tooltip = fairValueGapsExtendTooltip,       minval = 0)

showDailyLevelsInput            = input(        false,      'Daily',    group = LEVELS_GROUP,   inline = 'daily')
dailyLevelsStyleInput           = input.string( SOLID,      '',         group = LEVELS_GROUP,   inline = 'daily',   options = [SOLID,DASHED,DOTTED])
dailyLevelsColorInput           = input(        BLUE,       '',         group = LEVELS_GROUP,   inline = 'daily')
showWeeklyLevelsInput           = input(        false,      'Weekly',   group = LEVELS_GROUP,   inline = 'weekly')
weeklyLevelsStyleInput          = input.string( SOLID,      '',         group = LEVELS_GROUP,   inline = 'weekly',  options = [SOLID,DASHED,DOTTED])
weeklyLevelsColorInput          = input(        BLUE,       '',         group = LEVELS_GROUP,   inline = 'weekly')
showMonthlyLevelsInput          = input(        false,      'Monthly',   group = LEVELS_GROUP,   inline = 'monthly')
monthlyLevelsStyleInput         = input.string( SOLID,      '',         group = LEVELS_GROUP,   inline = 'monthly', options = [SOLID,DASHED,DOTTED])
monthlyLevelsColorInput         = input(        BLUE,       '',         group = LEVELS_GROUP,   inline = 'monthly')

showPremiumDiscountZonesInput   = input(        false,      'Premium/Discount Zones',   group = ZONES_GROUP , tooltip = showPremiumDiscountZonesTooltip)
premiumZoneColorInput           = input.color(  RED,        'Premium Zone',             group = ZONES_GROUP)
equilibriumZoneColorInput       = input.color(  GRAY,       'Equilibrium Zone',         group = ZONES_GROUP)
discountZoneColorInput          = input.color(  GREEN,      'Discount Zone',            group = ZONES_GROUP)

//---------------------------------------------------------------------------------------------------------------------}
//DATA STRUCTURES & VARIABLES
//---------------------------------------------------------------------------------------------------------------------{
// @type                            UDT representing alerts as bool fields
// @field internalBullishBOS        internal structure custom alert
// @field internalBearishBOS        internal structure custom alert
// @field internalBullishCHoCH      internal structure custom alert
// @field internalBearishCHoCH      internal structure custom alert
// @field swingBullishBOS           swing structure custom alert
// @field swingBearishBOS           swing structure custom alert
// @field swingBullishCHoCH         swing structure custom alert
// @field swingBearishCHoCH         swing structure custom alert
// @field internalBullishOrderBlock internal order block custom alert
// @field internalBearishOrderBlock internal order block custom alert
// @field swingBullishOrderBlock    swing order block custom alert
// @field swingBearishOrderBlock    swing order block custom alert
// @field equalHighs                equal high low custom alert
// @field equalLows                 equal high low custom alert
// @field bullishFairValueGap       fair value gap custom alert
// @field bearishFairValueGap       fair value gap custom alert
type alerts
    bool internalBullishBOS         = false
    bool internalBearishBOS         = false
    bool internalBullishCHoCH       = false
    bool internalBearishCHoCH       = false
    bool swingBullishBOS            = false
    bool swingBearishBOS            = false
    bool swingBullishCHoCH          = false
    bool swingBearishCHoCH          = false
    bool internalBullishOrderBlock  = false
    bool internalBearishOrderBlock  = false
    bool swingBullishOrderBlock     = false
    bool swingBearishOrderBlock     = false
    bool equalHighs                 = false
    bool equalLows                  = false
    bool bullishFairValueGap        = false
    bool bearishFairValueGap        = false

// @type                            UDT representing last swing extremes (top & bottom)
// @field top                       last top swing price
// @field bottom                    last bottom swing price
// @field barTime                   last swing bar time
// @field barIndex                  last swing bar index
// @field lastTopTime               last top swing time
// @field lastBottomTime            last bottom swing time
type trailingExtremes
    float top
    float bottom
    int barTime
    int barIndex
    int lastTopTime
    int lastBottomTime

// @type                            UDT representing Fair Value Gaps
// @field top                       top price
// @field bottom                    bottom price
// @field bias                      bias (BULLISH or BEARISH)
// @field topBox                    top box
// @field bottomBox                 bottom box
type fairValueGap
    float top
    float bottom
    int bias
    box topBox
    box bottomBox

// @type                            UDT representing trend bias
// @field bias                      BULLISH or BEARISH
type trend
    int bias    

// @type                            UDT representing Equal Highs Lows display
// @field l_ine                     displayed line
// @field l_abel                    displayed label
type equalDisplay
    line l_ine      = na
    label l_abel    = na

// @type                            UDT representing a pivot point (swing point) 
// @field currentLevel              current price level
// @field lastLevel                 last price level
// @field crossed                   true if price level is crossed
// @field barTime                   bar time
// @field barIndex                  bar index    
type pivot
    float currentLevel
    float lastLevel
    bool crossed
    int barTime     = time
    int barIndex    = bar_index

// @type                            UDT representing an order block
// @field barHigh                   bar high
// @field barLow                    bar low
// @field barTime                   bar time
// @field bias                      BULLISH or BEARISH
type orderBlock
    float barHigh
    float barLow
    int barTime    
    int bias

// @variable                        current swing pivot high    
var pivot swingHigh                 = pivot.new(na,na,false)
// @variable                        current swing pivot low
var pivot swingLow                  = pivot.new(na,na,false)
// @variable                        current internal pivot high
var pivot internalHigh              = pivot.new(na,na,false)
// @variable                        current internal pivot low
var pivot internalLow               = pivot.new(na,na,false)
// @variable                        current equal high pivot
var pivot equalHigh                 = pivot.new(na,na,false)
// @variable                        current equal low pivot
var pivot equalLow                  = pivot.new(na,na,false)
// @variable                        swing trend bias
var trend swingTrend                = trend.new(0)
// @variable                        internal trend bias
var trend internalTrend             = trend.new(0)
// @variable                        equal high display
var equalDisplay equalHighDisplay   = equalDisplay.new()
// @variable                        equal low display
var equalDisplay equalLowDisplay    = equalDisplay.new()
// @variable                        storage for fairValueGap UDTs
var array<fairValueGap> fairValueGaps = array.new<fairValueGap>()
// @variable                        storage for parsed highs
var array<float> parsedHighs        = array.new<float>()
// @variable                        storage for parsed lows
var array<float> parsedLows         = array.new<float>()
// @variable                        storage for raw highs
var array<float> highs              = array.new<float>()
// @variable                        storage for raw lows
var array<float> lows               = array.new<float>()
// @variable                        storage for bar time values
var array<int> times                = array.new<int>()
// @variable                        last trailing swing high and low
var trailingExtremes trailing       = trailingExtremes.new()
// @variable                                storage for orderBlock UDTs (swing order blocks)
var array<orderBlock> swingOrderBlocks      = array.new<orderBlock>()
// @variable                                storage for orderBlock UDTs (internal order blocks)
var array<orderBlock> internalOrderBlocks   = array.new<orderBlock>()
// @variable                                storage for swing order blocks boxes
var array<box> swingOrderBlocksBoxes        = array.new<box>()
// @variable                                storage for internal order blocks boxes
var array<box> internalOrderBlocksBoxes     = array.new<box>()
// @variable                        color for swing bullish structures
var swingBullishColor               = styleInput == MONOCHROME ? MONO_BULLISH : swingBullColorInput
// @variable                        color for swing bearish structures
var swingBearishColor               = styleInput == MONOCHROME ? MONO_BEARISH : swingBearColorInput
// @variable                        color for bullish fair value gaps
var fairValueGapBullishColor        = styleInput == MONOCHROME ? color.new(MONO_BULLISH,70) : fairValueGapsBullColorInput
// @variable                        color for bearish fair value gaps
var fairValueGapBearishColor        = styleInput == MONOCHROME ? color.new(MONO_BEARISH,70) : fairValueGapsBearColorInput
// @variable                        color for premium zone
var premiumZoneColor                = styleInput == MONOCHROME ? MONO_BEARISH : premiumZoneColorInput
// @variable                        color for discount zone
var discountZoneColor               = styleInput == MONOCHROME ? MONO_BULLISH : discountZoneColorInput 
// @variable                        bar index on current script iteration
varip int currentBarIndex           = bar_index
// @variable                        bar index on last script iteration
varip int lastBarIndex              = bar_index
// @variable                        alerts in current bar
alerts currentAlerts                = alerts.new()
// @variable                        time at start of chart
var initialTime                     = time

// we create the needed boxes for displaying order blocks at the first execution
if barstate.isfirst
    if showSwingOrderBlocksInput
        for index = 1 to swingOrderBlocksSizeInput
            swingOrderBlocksBoxes.push(box.new(na,na,na,na,xloc = xloc.bar_time,extend = extend.right))
    if showInternalOrderBlocksInput
        for index = 1 to internalOrderBlocksSizeInput
            internalOrderBlocksBoxes.push(box.new(na,na,na,na,xloc = xloc.bar_time,extend = extend.right))

// @variable                        source to use in bearish order blocks mitigation
bearishOrderBlockMitigationSource   = orderBlockMitigationInput == CLOSE ? close : high
// @variable                        source to use in bullish order blocks mitigation
bullishOrderBlockMitigationSource   = orderBlockMitigationInput == CLOSE ? close : low
// @variable                        default volatility measure
atrMeasure                          = ta.atr(200)
// @variable                        parsed volatility measure by user settings
volatilityMeasure                   = orderBlockFilterInput == ATR ? atrMeasure : ta.cum(ta.tr)/bar_index
// @variable                        true if current bar is a high volatility bar
highVolatilityBar                   = (high - low) >= (2 * volatilityMeasure)
// @variable                        parsed high
parsedHigh                          = highVolatilityBar ? low : high
// @variable                        parsed low
parsedLow                           = highVolatilityBar ? high : low

// we store current values into the arrays at each bar
parsedHighs.push(parsedHigh)
parsedLows.push(parsedLow)
highs.push(high)
lows.push(low)
times.push(time)

//---------------------------------------------------------------------------------------------------------------------}
//USER-DEFINED FUNCTIONS
//---------------------------------------------------------------------------------------------------------------------{
// @function            Get the value of the current leg, it can be 0 (bearish) or 1 (bullish)
// @returns             int
leg(int size) =>
    var leg     = 0    
    newLegHigh  = high[size] > ta.highest( size)
    newLegLow   = low[size]  < ta.lowest(  size)
    
    if newLegHigh
        leg := BEARISH_LEG
    else if newLegLow
        leg := BULLISH_LEG
    leg

// @function            Identify whether the current value is the start of a new leg (swing)
// @param leg           (int) Current leg value
// @returns             bool
startOfNewLeg(int leg)      => ta.change(leg) != 0

// @function            Identify whether the current level is the start of a new bearish leg (swing)
// @param leg           (int) Current leg value
// @returns             bool
startOfBearishLeg(int leg)  => ta.change(leg) == -1

// @function            Identify whether the current level is the start of a new bullish leg (swing)
// @param leg           (int) Current leg value
// @returns             bool
startOfBullishLeg(int leg)  => ta.change(leg) == +1

// @function            create a new label
// @param labelTime     bar time coordinate
// @param labelPrice    price coordinate
// @param tag           text to display
// @param labelColor    text color
// @param labelStyle    label style
// @returns             label ID
drawLabel(int labelTime, float labelPrice, string tag, color labelColor, string labelStyle) =>    
    var label l_abel = na

    if modeInput == PRESENT
        l_abel.delete()

    l_abel := label.new(chart.point.new(labelTime,na,labelPrice),tag,xloc.bar_time,color=color(na),textcolor=labelColor,style = labelStyle,size = size.small)

// @function            create a new line and label representing an EQH or EQL
// @param p_ivot        starting pivot
// @param level         price level of current pivot
// @param size          how many bars ago was the current pivot detected
// @param equalHigh     true for EQH, false for EQL
// @returns             label ID
drawEqualHighLow(pivot p_ivot, float level, int size, bool equalHigh) =>
    equalDisplay e_qualDisplay = equalHigh ? equalHighDisplay : equalLowDisplay
    
    string tag          = 'EQL'
    color equalColor    = swingBullishColor
    string labelStyle   = label.style_label_up

    if equalHigh
        tag         := 'EQH'
        equalColor  := swingBearishColor
        labelStyle  := label.style_label_down

    if modeInput == PRESENT
        line.delete(    e_qualDisplay.l_ine)
        label.delete(   e_qualDisplay.l_abel)
        
    e_qualDisplay.l_ine     := line.new(chart.point.new(p_ivot.barTime,na,p_ivot.currentLevel), chart.point.new(time[size],na,level), xloc = xloc.bar_time, color = equalColor, style = line.style_dotted)
    labelPosition           = math.round(0.5*(p_ivot.barIndex + bar_index - size))
    e_qualDisplay.l_abel    := label.new(chart.point.new(na,labelPosition,level), tag, xloc.bar_index, color = color(na), textcolor = equalColor, style = labelStyle, size = equalHighsLowsSizeInput)

// @function            store current structure and trailing swing points, and also display swing points and equal highs/lows
// @param size          (int) structure size
// @param equalHighLow  (bool) true for displaying current highs/lows
// @param internal      (bool) true for getting internal structures
// @returns             label ID
getCurrentStructure(int size,bool equalHighLow = false, bool internal = false) =>        
    currentLeg              = leg(size)
    newPivot                = startOfNewLeg(currentLeg)
    pivotLow                = startOfBullishLeg(currentLeg)
    pivotHigh               = startOfBearishLeg(currentLeg)

    if newPivot
        if pivotLow
            pivot p_ivot    = equalHighLow ? equalLow : internal ? internalLow : swingLow    

            if equalHighLow and math.abs(p_ivot.currentLevel - low[size]) < equalHighsLowsThresholdInput * atrMeasure                
                drawEqualHighLow(p_ivot, low[size], size, false)

            p_ivot.lastLevel    := p_ivot.currentLevel
            p_ivot.currentLevel := low[size]
            p_ivot.crossed      := false
            p_ivot.barTime      := time[size]
            p_ivot.barIndex     := bar_index[size]

            if not equalHighLow and not internal
                trailing.bottom         := p_ivot.currentLevel
                trailing.barTime        := p_ivot.barTime
                trailing.barIndex       := p_ivot.barIndex
                trailing.lastBottomTime := p_ivot.barTime

            if showSwingsInput and not internal and not equalHighLow
                drawLabel(time[size], p_ivot.currentLevel, p_ivot.currentLevel < p_ivot.lastLevel ? 'LL' : 'HL', swingBullishColor, label.style_label_up)            
        else
            pivot p_ivot = equalHighLow ? equalHigh : internal ? internalHigh : swingHigh

            if equalHighLow and math.abs(p_ivot.currentLevel - high[size]) < equalHighsLowsThresholdInput * atrMeasure
                drawEqualHighLow(p_ivot,high[size],size,true)                

            p_ivot.lastLevel    := p_ivot.currentLevel
            p_ivot.currentLevel := high[size]
            p_ivot.crossed      := false
            p_ivot.barTime      := time[size]
            p_ivot.barIndex     := bar_index[size]

            if not equalHighLow and not internal
                trailing.top            := p_ivot.currentLevel
                trailing.barTime        := p_ivot.barTime
                trailing.barIndex       := p_ivot.barIndex
                trailing.lastTopTime    := p_ivot.barTime

            if showSwingsInput and not internal and not equalHighLow
                drawLabel(time[size], p_ivot.currentLevel, p_ivot.currentLevel > p_ivot.lastLevel ? 'HH' : 'LH', swingBearishColor, label.style_label_down)
                
// @function                draw line and label representing a structure
// @param p_ivot            base pivot point
// @param tag               test to display
// @param structureColor    base color
// @param lineStyle         line style
// @param labelStyle        label style
// @param labelSize         text size
// @returns                 label ID
drawStructure(pivot p_ivot, string tag, color structureColor, string lineStyle, string labelStyle, string labelSize) =>    
    var line l_ine      = line.new(na,na,na,na,xloc = xloc.bar_time)
    var label l_abel    = label.new(na,na)

    if modeInput == PRESENT
        l_ine.delete()
        l_abel.delete()

    l_ine   := line.new(chart.point.new(p_ivot.barTime,na,p_ivot.currentLevel), chart.point.new(time,na,p_ivot.currentLevel), xloc.bar_time, color=structureColor, style=lineStyle)
    l_abel  := label.new(chart.point.new(na,math.round(0.5*(p_ivot.barIndex+bar_index)),p_ivot.currentLevel), tag, xloc.bar_index, color=color(na), textcolor=structureColor, style=labelStyle, size = labelSize)

// @function            delete order blocks
// @param internal      true for internal order blocks
// @returns             orderBlock ID
deleteOrderBlocks(bool internal = false) =>
    array<orderBlock> orderBlocks = internal ? internalOrderBlocks : swingOrderBlocks

    for [index,eachOrderBlock] in orderBlocks
        bool crossedOderBlock = false
        
        if bearishOrderBlockMitigationSource > eachOrderBlock.barHigh and eachOrderBlock.bias == BEARISH
            crossedOderBlock := true
            if internal
                currentAlerts.internalBearishOrderBlock := true
            else
                currentAlerts.swingBearishOrderBlock    := true
        else if bullishOrderBlockMitigationSource < eachOrderBlock.barLow and eachOrderBlock.bias == BULLISH
            crossedOderBlock := true
            if internal
                currentAlerts.internalBullishOrderBlock := true
            else
                currentAlerts.swingBullishOrderBlock    := true
        if crossedOderBlock                    
            orderBlocks.remove(index)            

// @function            fetch and store order blocks
// @param p_ivot        base pivot point
// @param internal      true for internal order blocks
// @param bias          BULLISH or BEARISH
// @returns             void
storeOrdeBlock(pivot p_ivot,bool internal = false,int bias) =>
    if (not internal and showSwingOrderBlocksInput) or (internal and showInternalOrderBlocksInput)

        array<float> a_rray = na
        int parsedIndex = na

        if bias == BEARISH
            a_rray      := parsedHighs.slice(p_ivot.barIndex,bar_index)
            parsedIndex := p_ivot.barIndex + a_rray.indexof(a_rray.max())  
        else
            a_rray      := parsedLows.slice(p_ivot.barIndex,bar_index)
            parsedIndex := p_ivot.barIndex + a_rray.indexof(a_rray.min())                        

        orderBlock o_rderBlock          = orderBlock.new(parsedHighs.get(parsedIndex), parsedLows.get(parsedIndex), times.get(parsedIndex),bias)
        array<orderBlock> orderBlocks   = internal ? internalOrderBlocks : swingOrderBlocks
        
        if orderBlocks.size() >= 100
            orderBlocks.pop()
        orderBlocks.unshift(o_rderBlock)

// @function            draw order blocks as boxes
// @param internal      true for internal order blocks
// @returns             void
drawOrderBlocks(bool internal = false) =>        
    array<orderBlock> orderBlocks = internal ? internalOrderBlocks : swingOrderBlocks
    orderBlocksSize = orderBlocks.size()

    if orderBlocksSize > 0        
        maxOrderBlocks                      = internal ? internalOrderBlocksSizeInput : swingOrderBlocksSizeInput
        array<orderBlock> parsedOrdeBlocks  = orderBlocks.slice(0, math.min(maxOrderBlocks,orderBlocksSize))
        array<box> b_oxes                   = internal ? internalOrderBlocksBoxes : swingOrderBlocksBoxes        

        for [index,eachOrderBlock] in parsedOrdeBlocks
            orderBlockColor = styleInput == MONOCHROME ? (eachOrderBlock.bias == BEARISH ? color.new(MONO_BEARISH,80) : color.new(MONO_BULLISH,80)) : internal ? (eachOrderBlock.bias == BEARISH ? internalBearishOrderBlockColor : internalBullishOrderBlockColor) : (eachOrderBlock.bias == BEARISH ? swingBearishOrderBlockColor : swingBullishOrderBlockColor)

            box b_ox        = b_oxes.get(index)
            b_ox.set_top_left_point(    chart.point.new(eachOrderBlock.barTime,na,eachOrderBlock.barHigh))
            b_ox.set_bottom_right_point(chart.point.new(last_bar_time,na,eachOrderBlock.barLow))        
            b_ox.set_border_color(      internal ? na : orderBlockColor)
            b_ox.set_bgcolor(           orderBlockColor)

// @function            detect and draw structures, also detect and store order blocks
// @param internal      true for internal structures or order blocks
// @returns             void
displayStructure(bool internal = false) =>
    var bullishBar = true
    var bearishBar = true

    if internalFilterConfluenceInput
        bullishBar := high - math.max(close, open) > math.min(close, open - low)
        bearishBar := high - math.max(close, open) < math.min(close, open - low)
    
    pivot p_ivot    = internal ? internalHigh : swingHigh
    trend t_rend    = internal ? internalTrend : swingTrend

    lineStyle       = internal ? line.style_dashed : line.style_solid
    labelSize       = internal ? internalStructureSize : swingStructureSize

    extraCondition  = internal ? internalHigh.currentLevel != swingHigh.currentLevel and bullishBar : true
    bullishColor    = styleInput == MONOCHROME ? MONO_BULLISH : internal ? internalBullColorInput : swingBullColorInput

    if ta.crossover(close,p_ivot.currentLevel) and not p_ivot.crossed and extraCondition
        string tag = t_rend.bias == BEARISH ? CHOCH : BOS

        if internal
            currentAlerts.internalBullishCHoCH  := tag == CHOCH
            currentAlerts.internalBullishBOS    := tag == BOS
        else
            currentAlerts.swingBullishCHoCH     := tag == CHOCH
            currentAlerts.swingBullishBOS       := tag == BOS

        p_ivot.crossed  := true
        t_rend.bias     := BULLISH

        displayCondition = internal ? showInternalsInput and (showInternalBullInput == ALL or (showInternalBullInput == BOS and tag != CHOCH) or (showInternalBullInput == CHOCH and tag == CHOCH)) : showStructureInput and (showSwingBullInput == ALL or (showSwingBullInput == BOS and tag != CHOCH) or (showSwingBullInput == CHOCH and tag == CHOCH))

        if displayCondition                        
            drawStructure(p_ivot,tag,bullishColor,lineStyle,label.style_label_down,labelSize)

        if (internal and showInternalOrderBlocksInput) or (not internal and showSwingOrderBlocksInput)
            storeOrdeBlock(p_ivot,internal,BULLISH)

    p_ivot          := internal ? internalLow : swingLow    
    extraCondition  := internal ? internalLow.currentLevel != swingLow.currentLevel and bearishBar : true
    bearishColor    = styleInput == MONOCHROME ? MONO_BEARISH : internal ? internalBearColorInput : swingBearColorInput

    if ta.crossunder(close,p_ivot.currentLevel) and not p_ivot.crossed and extraCondition
        string tag = t_rend.bias == BULLISH ? CHOCH : BOS

        if internal
            currentAlerts.internalBearishCHoCH  := tag == CHOCH
            currentAlerts.internalBearishBOS    := tag == BOS
        else
            currentAlerts.swingBearishCHoCH     := tag == CHOCH
            currentAlerts.swingBearishBOS       := tag == BOS

        p_ivot.crossed := true
        t_rend.bias := BEARISH

        displayCondition = internal ? showInternalsInput and (showInternalBearInput == ALL or (showInternalBearInput == BOS and tag != CHOCH) or (showInternalBearInput == CHOCH and tag == CHOCH)) : showStructureInput and (showSwingBearInput == ALL or (showSwingBearInput == BOS and tag != CHOCH) or (showSwingBearInput == CHOCH and tag == CHOCH))
        
        if displayCondition                        
            drawStructure(p_ivot,tag,bearishColor,lineStyle,label.style_label_up,labelSize)

        if (internal and showInternalOrderBlocksInput) or (not internal and showSwingOrderBlocksInput)
            storeOrdeBlock(p_ivot,internal,BEARISH)

// @function            draw one fair value gap box (each fair value gap has two boxes)
// @param leftTime      left time coordinate
// @param rightTime     right time coordinate
// @param topPrice      top price level
// @param bottomPrice   bottom price level
// @param boxColor      box color
// @returns             box ID
fairValueGapBox(leftTime,rightTime,topPrice,bottomPrice,boxColor) => box.new(chart.point.new(leftTime,na,topPrice),chart.point.new(rightTime + fairValueGapsExtendInput * (time-time[1]),na,bottomPrice), xloc=xloc.bar_time, border_color = boxColor, bgcolor = boxColor)

// @function            delete fair value gaps
// @returns             fairValueGap ID
deleteFairValueGaps() =>
    for [index,eachFairValueGap] in fairValueGaps
        if (low < eachFairValueGap.bottom and eachFairValueGap.bias == BULLISH) or (high > eachFairValueGap.top and eachFairValueGap.bias == BEARISH)
            eachFairValueGap.topBox.delete()
            eachFairValueGap.bottomBox.delete()
            fairValueGaps.remove(index)
    
// @function            draw fair value gaps
// @returns             fairValueGap ID
drawFairValueGaps() => 
    [lastClose, lastOpen, lastTime, currentHigh, currentLow, currentTime, last2High, last2Low] = request.security(syminfo.tickerid, fairValueGapsTimeframeInput, [close[1], open[1], time[1], high[0], low[0], time[0], high[2], low[2]],lookahead = barmerge.lookahead_on)

    barDeltaPercent     = (lastClose - lastOpen) / (lastOpen * 100)
    newTimeframe        = timeframe.change(fairValueGapsTimeframeInput)
    threshold           = fairValueGapsThresholdInput ? ta.cum(math.abs(newTimeframe ? barDeltaPercent : 0)) / bar_index * 2 : 0

    bullishFairValueGap = currentLow > last2High and lastClose > last2High and barDeltaPercent > threshold and newTimeframe
    bearishFairValueGap = currentHigh < last2Low and lastClose < last2Low and -barDeltaPercent > threshold and newTimeframe

    if bullishFairValueGap
        currentAlerts.bullishFairValueGap := true
        fairValueGaps.unshift(fairValueGap.new(currentLow,last2High,BULLISH,fairValueGapBox(lastTime,currentTime,currentLow,math.avg(currentLow,last2High),fairValueGapBullishColor),fairValueGapBox(lastTime,currentTime,math.avg(currentLow,last2High),last2High,fairValueGapBullishColor)))
    if bearishFairValueGap
        currentAlerts.bearishFairValueGap := true
        fairValueGaps.unshift(fairValueGap.new(currentHigh,last2Low,BEARISH,fairValueGapBox(lastTime,currentTime,currentHigh,math.avg(currentHigh,last2Low),fairValueGapBearishColor),fairValueGapBox(lastTime,currentTime,math.avg(currentHigh,last2Low),last2Low,fairValueGapBearishColor)))

// @function            get line style from string
// @param style         line style
// @returns             string
getStyle(string style) =>
    switch style
        SOLID => line.style_solid
        DASHED => line.style_dashed
        DOTTED => line.style_dotted

// @function            draw MultiTimeFrame levels
// @param timeframe     base timeframe
// @param sameTimeframe true if chart timeframe is same as base timeframe
// @param style         line style
// @param levelColor    line and text color
// @returns             void
drawLevels(string timeframe, bool sameTimeframe, string style, color levelColor) =>
    [topLevel, bottomLevel, leftTime, rightTime] = request.security(syminfo.tickerid, timeframe, [high[1], low[1], time[1], time],lookahead = barmerge.lookahead_on)

    float parsedTop         = sameTimeframe ? high : topLevel
    float parsedBottom      = sameTimeframe ? low : bottomLevel    

    int parsedLeftTime      = sameTimeframe ? time : leftTime
    int parsedRightTime     = sameTimeframe ? time : rightTime

    int parsedTopTime       = time
    int parsedBottomTime    = time

    if not sameTimeframe
        int leftIndex               = times.binary_search_rightmost(parsedLeftTime)
        int rightIndex              = times.binary_search_rightmost(parsedRightTime)

        array<int> timeArray        = times.slice(leftIndex,rightIndex)
        array<float> topArray       = highs.slice(leftIndex,rightIndex)
        array<float> bottomArray    = lows.slice(leftIndex,rightIndex)

        parsedTopTime               := timeArray.size() > 0 ? timeArray.get(topArray.indexof(topArray.max())) : initialTime
        parsedBottomTime            := timeArray.size() > 0 ? timeArray.get(bottomArray.indexof(bottomArray.min())) : initialTime

    var line topLine        = line.new(na, na, na, na, xloc = xloc.bar_time, color = levelColor, style = getStyle(style))
    var line bottomLine     = line.new(na, na, na, na, xloc = xloc.bar_time, color = levelColor, style = getStyle(style))
    var label topLabel      = label.new(na, na, xloc = xloc.bar_time, text = str.format('P{0}H',timeframe), color=color(na), textcolor = levelColor, size = size.small, style = label.style_label_left)
    var label bottomLabel   = label.new(na, na, xloc = xloc.bar_time, text = str.format('P{0}L',timeframe), color=color(na), textcolor = levelColor, size = size.small, style = label.style_label_left)

    topLine.set_first_point(    chart.point.new(parsedTopTime,na,parsedTop))
    topLine.set_second_point(   chart.point.new(last_bar_time + 20 * (time-time[1]),na,parsedTop))   
    topLabel.set_point(         chart.point.new(last_bar_time + 20 * (time-time[1]),na,parsedTop))

    bottomLine.set_first_point( chart.point.new(parsedBottomTime,na,parsedBottom))    
    bottomLine.set_second_point(chart.point.new(last_bar_time + 20 * (time-time[1]),na,parsedBottom))
    bottomLabel.set_point(      chart.point.new(last_bar_time + 20 * (time-time[1]),na,parsedBottom))

// @function            true if chart timeframe is higher than provided timeframe
// @param timeframe     timeframe to check
// @returns             bool
higherTimeframe(string timeframe) => timeframe.in_seconds() > timeframe.in_seconds(timeframe)

// @function            update trailing swing points
// @returns             int
updateTrailingExtremes() =>
    trailing.top            := math.max(high,trailing.top)
    trailing.lastTopTime    := trailing.top == high ? time : trailing.lastTopTime
    trailing.bottom         := math.min(low,trailing.bottom)
    trailing.lastBottomTime := trailing.bottom == low ? time : trailing.lastBottomTime

// @function            draw trailing swing points
// @returns             void
drawHighLowSwings() =>
    var line topLine        = line.new(na, na, na, na, color = swingBearishColor, xloc = xloc.bar_time)
    var line bottomLine     = line.new(na, na, na, na, color = swingBullishColor, xloc = xloc.bar_time)
    var label topLabel      = label.new(na, na, color=color(na), textcolor = swingBearishColor, xloc = xloc.bar_time, style = label.style_label_down, size = size.tiny)
    var label bottomLabel   = label.new(na, na, color=color(na), textcolor = swingBullishColor, xloc = xloc.bar_time, style = label.style_label_up, size = size.tiny)

    rightTimeBar            = last_bar_time + 20 * (time - time[1])

    topLine.set_first_point(    chart.point.new(trailing.lastTopTime, na, trailing.top))
    topLine.set_second_point(   chart.point.new(rightTimeBar, na, trailing.top))
    topLabel.set_point(         chart.point.new(rightTimeBar, na, trailing.top))
    topLabel.set_text(          swingTrend.bias == BEARISH ? 'Strong High' : 'Weak High')

    bottomLine.set_first_point( chart.point.new(trailing.lastBottomTime, na, trailing.bottom))
    bottomLine.set_second_point(chart.point.new(rightTimeBar, na, trailing.bottom))
    bottomLabel.set_point(      chart.point.new(rightTimeBar, na, trailing.bottom))
    bottomLabel.set_text(       swingTrend.bias == BULLISH ? 'Strong Low' : 'Weak Low')

// @function            draw a zone with a label and a box
// @param labelLevel    price level for label
// @param labelIndex    bar index for label
// @param top           top price level for box
// @param bottom        bottom price level for box
// @param tag           text to display
// @param zoneColor     base color
// @param style         label style
// @returns             void
drawZone(float labelLevel, int labelIndex, float top, float bottom, string tag, color zoneColor, string style) =>
    var label l_abel    = label.new(na,na,text = tag, color=color(na),textcolor = zoneColor, style = style, size = size.small)
    var box b_ox        = box.new(na,na,na,na,bgcolor = color.new(zoneColor,80),border_color = color(na), xloc = xloc.bar_time)

    b_ox.set_top_left_point(    chart.point.new(trailing.barTime,na,top))
    b_ox.set_bottom_right_point(chart.point.new(last_bar_time,na,bottom))

    l_abel.set_point(           chart.point.new(na,labelIndex,labelLevel))

// @function            draw premium/discount zones
// @returns             void
drawPremiumDiscountZones() =>
    drawZone(trailing.top, math.round(0.5*(trailing.barIndex + last_bar_index)), trailing.top, 0.95*trailing.top + 0.05*trailing.bottom, 'Premium', premiumZoneColor, label.style_label_down)

    equilibriumLevel = math.avg(trailing.top, trailing.bottom)
    drawZone(equilibriumLevel, last_bar_index, 0.525*trailing.top + 0.475*trailing.bottom, 0.525*trailing.bottom + 0.475*trailing.top, 'Equilibrium', equilibriumZoneColorInput, label.style_label_left)

    drawZone(trailing.bottom, math.round(0.5*(trailing.barIndex + last_bar_index)), 0.95*trailing.bottom + 0.05*trailing.top, trailing.bottom, 'Discount', discountZoneColor, label.style_label_up)

//---------------------------------------------------------------------------------------------------------------------}
//MUTABLE VARIABLES & EXECUTION
//---------------------------------------------------------------------------------------------------------------------{
parsedOpen  = showTrendInput ? open : na
candleColor = internalTrend.bias == BULLISH ? swingBullishColor : swingBearishColor
plotcandle(parsedOpen,high,low,close,color = candleColor, wickcolor = candleColor, bordercolor = candleColor)

if showHighLowSwingsInput or showPremiumDiscountZonesInput
    updateTrailingExtremes()

    if showHighLowSwingsInput
        drawHighLowSwings()

    if showPremiumDiscountZonesInput
        drawPremiumDiscountZones()

if showFairValueGapsInput
    deleteFairValueGaps()

getCurrentStructure(swingsLengthInput,false)
getCurrentStructure(5,false,true)

if showEqualHighsLowsInput
    getCurrentStructure(equalHighsLowsLengthInput,true)

if showInternalsInput or showInternalOrderBlocksInput or showTrendInput
    displayStructure(true)

if showStructureInput or showSwingOrderBlocksInput or showHighLowSwingsInput
    displayStructure()

if showInternalOrderBlocksInput
    deleteOrderBlocks(true)

if showSwingOrderBlocksInput
    deleteOrderBlocks()

if showFairValueGapsInput
    drawFairValueGaps()

if barstate.islastconfirmedhistory or barstate.islast
    if showInternalOrderBlocksInput        
        drawOrderBlocks(true)
        
    if showSwingOrderBlocksInput        
        drawOrderBlocks()

lastBarIndex    := currentBarIndex
currentBarIndex := bar_index
newBar          = currentBarIndex != lastBarIndex

if barstate.islastconfirmedhistory or (barstate.isrealtime and newBar)
    if showDailyLevelsInput and not higherTimeframe('D')
        drawLevels('D',timeframe.isdaily,dailyLevelsStyleInput,dailyLevelsColorInput)

    if showWeeklyLevelsInput and not higherTimeframe('W')
        drawLevels('W',timeframe.isweekly,weeklyLevelsStyleInput,weeklyLevelsColorInput)

    if showMonthlyLevelsInput and not higherTimeframe('M')
        drawLevels('M',timeframe.ismonthly,monthlyLevelsStyleInput,monthlyLevelsColorInput)

//---------------------------------------------------------------------------------------------------------------------}
//ALERTS
//---------------------------------------------------------------------------------------------------------------------{
alertcondition(currentAlerts.internalBullishBOS,        'Internal Bullish BOS',         'Internal Bullish BOS formed')
alertcondition(currentAlerts.internalBullishCHoCH,      'Internal Bullish CHoCH',       'Internal Bullish CHoCH formed')
alertcondition(currentAlerts.internalBearishBOS,        'Internal Bearish BOS',         'Internal Bearish BOS formed')
alertcondition(currentAlerts.internalBearishCHoCH,      'Internal Bearish CHoCH',       'Internal Bearish CHoCH formed')

alertcondition(currentAlerts.swingBullishBOS,           'Bullish BOS',                  'Internal Bullish BOS formed')
alertcondition(currentAlerts.swingBullishCHoCH,         'Bullish CHoCH',                'Internal Bullish CHoCH formed')
alertcondition(currentAlerts.swingBearishBOS,           'Bearish BOS',                  'Bearish BOS formed')
alertcondition(currentAlerts.swingBearishCHoCH,         'Bearish CHoCH',                'Bearish CHoCH formed')

alertcondition(currentAlerts.internalBullishOrderBlock, 'Bullish Internal OB Breakout', 'Price broke bullish internal OB')
alertcondition(currentAlerts.internalBearishOrderBlock, 'Bearish Internal OB Breakout', 'Price broke bearish internal OB')
alertcondition(currentAlerts.swingBullishOrderBlock,    'Bullish Swing OB Breakout',    'Price broke bullish swing OB')
alertcondition(currentAlerts.swingBearishOrderBlock,    'Bearish Swing OB Breakout',    'Price broke bearish swing OB')

alertcondition(currentAlerts.equalHighs,                'Equal Highs',                  'Equal highs detected')
alertcondition(currentAlerts.equalLows,                 'Equal Lows',                   'Equal lows detected')

alertcondition(currentAlerts.bullishFairValueGap,       'Bullish FVG',                  'Bullish FVG formed')
alertcondition(currentAlerts.bearishFairValueGap,       'Bearish FVG',                  'Bearish FVG formed')

//---------------------------------------------------------------------------------------------------------------------}
