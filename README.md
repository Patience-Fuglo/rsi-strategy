# RSI Trading Bot with Alpaca API

This project implements a **Relative Strength Index (RSI)**-based stock trading bot in Python. It walks through the entire process from setting up the environment, calculating the RSI, backtesting the strategy, plotting results, and deploying the algorithm live using the Alpaca trading API.

---

## Strategy Overview

The RSI Overbought/Oversold Strategy is a popular momentum trading approach using the RSI indicator:

* **RSI Calculation:** Uses a 14-day period (customizable).
* **Overbought Threshold:** Typically set at 70.
* **Oversold Threshold:** Typically set at 30.
* **Buy Signal:** When RSI crosses below 30 and then back above (indicating oversold).
* **Sell Signal:** When RSI crosses above 70 (indicating overbought).

In this project, we apply the strategy on the QQQ ETF but you can test other tickers.

---

## Table of Contents

1. [Setting up the environment](#setting-up-the-environment)
2. [Importing libraries & fetching historical data](#importing-libraries--fetching-historical-data)
3. [Calculating the RSI](#calculating-the-rsi)
4. [Implementing the trading strategy](#implementing-the-trading-strategy)
5. [Backtesting & comparing with SPY](#backtesting--comparing-with-spy)
6. [Plotting the results](#plotting-the-results)
7. [Connecting to Alpaca](#connecting-to-alpaca)
8. [Implementing live trading](#implementing-live-trading)
9. [Running the algorithm](#running-the-algorithm)

---

## Setting up the environment

Install the required libraries:

```bash
pip install pandas numpy matplotlib yfinance alpaca-trade-api
```

---

## Importing libraries & fetching historical data

```python
import numpy as np
import pandas as pd
import yfinance as yf
import matplotlib.pyplot as plt

symbol = 'QQQ'
start_date = '2015-01-01'
end_date = '2022-12-31'

data = yf.download(symbol, start=start_date, end=end_date)
```

---

## Calculating the RSI

```python
def rsi(data, period=14):
    delta = data.diff().dropna()
    gain = delta.where(delta > 0, 0)
    loss = -delta.where(delta < 0, 0)
    avg_gain = gain.rolling(window=period).mean()
    avg_loss = loss.rolling(window=period).mean()
    rs = avg_gain / avg_loss
    return 100 - (100 / (1 + rs))

data['RSI'] = rsi(data['Close'])
```

---

## Implementing the trading strategy

```python
data['Signal'] = 0
data.loc[data['RSI'] < 30, 'Signal'] = 1     # Buy
data.loc[data['RSI'] > 70, 'Signal'] = -1    # Sell
```

---

## Backtesting & comparing with SPY

```python
data['Daily_Return'] = data['Close'].pct_change()
data['Strategy_Return'] = data['Daily_Return'] * data['Signal'].shift(1)
data['Cumulative_Return'] = (1 + data['Strategy_Return']).cumprod()

spy_data = yf.download('SPY', start=start_date, end=end_date)
spy_data['Daily_Return'] = spy_data['Close'].pct_change()
spy_data['Cumulative_Return'] = (1 + spy_data['Daily_Return']).cumprod()
```

---

## Plotting the results

```python
plt.figure(figsize=(12, 6))
plt.plot(data.index, data['Cumulative_Return'], label='RSI Strategy')
plt.plot(spy_data.index, spy_data['Cumulative_Return'], label='SPY')
plt.xlabel('Date')
plt.ylabel('Cumulative Returns')
plt.legend()
plt.show()
```

---

## Connecting to Alpaca

Sign up on [Alpaca](https://alpaca.markets), get your API keys, then:

```python
from alpaca_trade_api import REST

api_key = 'YOUR_API_KEY'
api_secret = 'YOUR_SECRET_KEY'
base_url = 'https://paper-api.alpaca.markets'  # Paper trading URL

api = REST(api_key, api_secret, base_url)
```

---

## Implementing live trading

```python
import time

def check_positions(symbol):
    positions = api.list_positions()
    for position in positions:
        if position.symbol == symbol:
            return int(position.qty)
    return 0

def trade(symbol, qty):
    current_rsi = rsi(yf.download(symbol, start=start_date, end=end_date, interval='1d')['Close'], 14).iloc[-1]
    position_qty = check_positions(symbol)
    
    if current_rsi < 30 and position_qty == 0:
        api.submit_order(symbol=symbol, qty=qty, side='buy', type='market', time_in_force='gtc')
        print(f"Buy order placed for {symbol}")
    elif current_rsi > 70 and position_qty > 0:
        api.submit_order(symbol=symbol, qty=position_qty, side='sell', type='market', time_in_force='gtc')
        print(f"Sell order placed for {symbol}")
    else:
        print(f"Holding {symbol}")
```

---

## Running the algorithm

```python
symbol = 'QQQ'
qty = 10

while True:
    trade(symbol, qty)
    time.sleep(86400)  # Check once per day (adjust as needed)
```

