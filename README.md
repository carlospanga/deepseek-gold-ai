# deepseek-gold-ai
trading robot ai

# DeepSeek Gold EA

A MetaTrader 4 Expert Advisor (EA) for trading gold (XAUUSD) based on the DeepSeek strategy.

## Features
- **Trading Window:** 4:00 AM - 4:45 AM EAT (UTC+3)
- **Strategy:** EMA Crossover (20/50), RSI Filter, ATR-based Volatility Filter
- **Risk Management:** 2% risk per trade, ATR-based stop loss and take profit

## Installation
1. Download the `DeepSeek.mq4` file.
2. Place it in the `MQL4/Experts` folder of your MetaTrader 4 installation.
3. Compile the EA in MetaTrader 4.
4. Attach it to the XAUUSD chart.

## Parameters
- **RiskPercent:** Risk percentage per trade (default: 2.0)
- **EMAFastPeriod:** Fast EMA period (default: 20)
- **EMASlowPeriod:** Slow EMA period (default: 50)
- **RSIPeriod:** RSI period (default: 14)
- **ATRPeriod:** ATR period (default: 14)
- **GMT_Offset:** Timezone offset for EAT (default: 3)

## Full EA Code
```mq4
//+------------------------------------------------------------------+
//|                                                      DeepSeek.mq4 |
//|                        Copyright 2023, DeepSeek Financial Technologies |
//|                                             https://www.deepseek.ai |
//+------------------------------------------------------------------+
#property copyright "DeepSeek Gold EA"
#property version   "1.00"
#property strict

// Strategy Parameters
input double RiskPercent    = 2.0;    // Risk % per Trade
input int    EMAFastPeriod  = 20;     // Fast EMA Period
input int    EMASlowPeriod  = 50;     // Slow EMA Period
input int    RSIPeriod      = 14;     // RSI Period
input int    ATRPeriod      = 14;     // ATR Period
input int    GMT_Offset     = 3;      // EAT Timezone Offset (UTC+3)

// Global Variables
datetime lastTradeTime;
double   atrValue;

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit()
  {
   return(INIT_SUCCEEDED);
  }

//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
  {
  }

//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
  {
   // Check trading time (4:00-4:45 AM EAT)
   if(!IsTradingTime()) return;

   // Check new bar
   if(Time[0] == lastTradeTime) return;

   // Calculate indicators
   double emaFast = iMA(NULL,0,EMAFastPeriod,0,MODE_EMA,PRICE_CLOSE,0);
   double emaSlow = iMA(NULL,0,EMASlowPeriod,0,MODE_EMA,PRICE_CLOSE,0);
   double rsi     = iRSI(NULL,0,RSIPeriod,PRICE_CLOSE,0);
   atrValue       = iATR(NULL,0,ATRPeriod,0);

   // Strategy Conditions
   bool emaCross    = (emaFast > emaSlow);
   bool rsiCondition= (rsi > 45);
   bool volatility  = (atrValue < (iATR(NULL,0,ATRPeriod,24) * 0.7));
   bool priceAction = (Close[1] > Open[1] && Close[0] > High[1]);

   if(emaCross && rsiCondition && volatility && priceAction)
     {
      EnterTrade();
      lastTradeTime = Time[0];
     }
  }

//+------------------------------------------------------------------+
//| Trading Time Validation                                          |
//+------------------------------------------------------------------+
bool IsTradingTime()
  {
   datetime serverTime = TimeCurrent();
   datetime localTime  = serverTime + (GMT_Offset * 3600);
   int hour            = TimeHour(localTime);
   int minute          = TimeMinute(localTime);

   // 4:00-4:45 AM EAT
   if((hour == 4 && minute >= 0) || (hour == 4 && minute <= 45))
      return true;

   return false;
  }

//+------------------------------------------------------------------+
//| Trade Entry Logic                                                |
//+------------------------------------------------------------------+
void EnterTrade()
  {
   double riskAmount  = AccountBalance() * (RiskPercent / 100);
   double tickValue   = MarketInfo(Symbol(), MODE_TICKVALUE);
   double stopLoss    = atrValue * 2;
   double takeProfit  = atrValue * 4;
   
   if(tickValue == 0) return;

   // Calculate position size
   double lots = (riskAmount / stopLoss) / tickValue;
   lots = NormalizeDouble(lots, 2);

   // Execute buy order
   int ticket = OrderSend(Symbol(), OP_BUY, lots, Ask, 3, 
                         Bid - (stopLoss * Point), 
                         Ask + (takeProfit * Point), 
                         "DeepSeek Gold Trade", 0, clrGreen);

   if(ticket < 0)
      Print("Order Failed. Error: ", GetLastError());
  }

//+------------------------------------------------------------------+
