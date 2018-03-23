# BigDataFinance
Trading Algorithm Based on VIX/Volatility Spread

from quantopian.pipeline import Pipeline
from quantopian.algorithm import attach_pipeline, pipeline_output
from quantopian.pipeline.factors import CustomFactor
from quantopian.pipeline.data.quandl import cboe_vix
import numpy as np
import math

class VIXFactor(CustomFactor):
  inputs = [cboe_vix.vix_close]
  window_length = 22
  
  def compute(self, today, assets, out, vix):
    vixout = vix[-22]
    out[:] = vixout
   
def initialize(context):
  pipe = Pipeline()
  attach_pipeline(pipe, 'vixtest')
  
  pipe.add(VIXFactor(), 'vix')
  
  schedule_function(func=rebalance,
                    date_rule=date_rules.month_start(days_offset=0),
                    time_rule=time_rules.market_open(hours=0,minutes=30),
                    half_dates=True)
                    
  set_slippage(slippage.FixedSlippage(spread=0.01)) #average $0.01 bud/ask spread per share
  set_commission(commission.PerShare(cost=0.005, min_trade_cost=1.00)) #$0.005/share, $1 minimum
  
def before_trading_start(context, data):
  context.output = pipeline_output('vixtest')
  context.stocks = context.output.iloc[0]
  context.asset1 = symbol('SPY')
  
def rebalance(context,data):
  spx_prices = data.history(context.asset1, "price", 23, "1d")
  spx_prices = spx_prices[0:22}
  spx_returns = spx_prices.pct_change().dropna().values
  spx_volatility = np.std(spx_returns,axis=0) * math.sqrt(252) * 100
  spx_cmreturn = np.cumprod(1+spx_returns)
  spx_cmreturn = spx_cmreturn[-1] - 1
  log.info(spx_cmreturn)
  
  for stock in context.stocks.index:
    if (context.stocks['vix'] <= spx_volatility) and (spx_cmreturn <0):
      order_target_percent(context.asset1,-1)
    else:
      order_target_percent(context.asset1,1)
