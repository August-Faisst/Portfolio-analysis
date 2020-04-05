# Portfolio-analysis

## About

This program is written in Python and is meant for analysing mainly risk and return for indiviual stocks as well as portfolios. It uses real prices data from the Yahoo Finance server and retrieves income statements and balances from Financial Modeling Prep. The program calculates a bunch of different key idicators for the given stock or portfolio such as risk, return, beta value, alpha values, correlations etc.

## How it works

First of all the program needs to know the ticker (symbol) of the stock you want to examine, or if it's a portfolio, you will have to provide both tickers of all the stocks in the portfolio and the weights of each stock in the portfolio. Secondly you will need to define the ticker for the benchemark (e.g. an index). Thirdly the program needs a timeframe in Datetime format e.g. (2015, 1, 1). Fourthly you should be aware of the risk free return (RF), which by default is set to 0.013 (1.3%). The RF is used later in the calculation of the required return (according to CAPM).
```python
# The given ticker and the benchmark (market) to compare
ticker = 'AAPL'
benchmark = '^GSPC'

# Risk free return
RF_return = 0.013

# Portfolio tickers and weights
tickers = ['TSLA', 'FB', 'AAPL', 'AMZN', 'GOOG']
wts = [0.2, 0.1, 0.1, 0.4, 0.2]

# Timeframe
start = dt.datetime(2015, 3, 1)
end = dt.datetime(2020, 3, 1)
```
The first function in the program is just a quick visualization of the price evolution for the given ticker. It returns a candlestick-diagram and a 100 days moving average plot (the number of days can be ajusted by .rolling(window=100).mean()). 
```python
def candlestick():
    df = web.DataReader(ticker, 'yahoo', start, end)

    # 100 moving average: takes the past 99 days and creates an average of these
    # prices and the current day's price
    #   Formula: sum of prices of the last 100 days / 100
    df['100ma'] = df['Adj Close'].rolling(window=100).mean()

    # Plotting the data in a 2 by 1 grid
    ax1 = plt.subplot2grid((6,1), (0,0), rowspan = 5, colspan = 1)
    plt.grid(color = '#D3D3D3', linestyle = '--', linewidth = 0.5)
    plt.yticks(fontsize = 8, fontname = font)
    plt.xticks(color = 'w')
    plt.title(ticker.replace('^',''), loc='center')
    ax2 = plt.subplot2grid((6,1), (5,0), rowspan=1, colspan=1, sharex=ax1)
    plt.yticks(fontsize = 8, fontname = font)
    ax1.xaxis_date()

    ax1.plot(df.index, df['100ma'], color = '#188fff', linewidth = 1)
    plt.xticks(fontsize = 8, fontname = font)

    # Resample and converting dates to mdates
    df_ohlc = df['Adj Close'].resample('10D').ohlc()
    df_volume = df['Volume'].resample('10D').sum()

    df_ohlc.reset_index(inplace=True)
    df_ohlc['Date'] = df_ohlc['Date'].map(mdates.date2num)
    #print(df_ohlc.tail())

    # Creating candlestick-diagram
    candlestick_ohlc(ax1, df_ohlc.values, width=5, colorup='g')
    ax2.fill_between(df_volume.index.map(mdates.date2num), df_volume.values, 0, color = '#188fff')

    plt.show()

candlestick()
```
The candlestick-diagram will look like this. Notice that also the volume is added in the lower part of the figure.
![Candlestick](https://user-images.githubusercontent.com/63104057/78458735-e582ce80-76b3-11ea-99db-994df12dcfb6.png)

The next function takes the tickers and weights in the portfolio and grabs the price data from Yahoo. Then it uses .resample('M') to reduce the amount of data to only monthly price data indstead of daily (you can change the 'M' to other presets such as 'W' for weekly). Futhermore we're not really interested in the raw price data, but rather in the returns of the stocks, which is why we calculate the monthly returns using .pct_chance().
With the monthly returns known the function calculates alot of different values: standard deviation (risk), mean (expected return), correlation coefficients, beta values, alpha values and required return. Also you will get a graf that shows the price evolution in relative terms for each ticker as well as a heatmap visualizing the correlations between the stocks in the portfolio. 
The function does also return some key financial highlights about the stocks in the portfolio. 
```python
def portfolio_analysis():
    df_tickers = web.DataReader(tickers, 'yahoo', start, end)
    df_tickers = df_tickers['Adj Close'].rename_axis(index = None, columns = None)
    df_benchmark = web.DataReader(benchmark, 'yahoo', start, end)
    df_benchmark = df_benchmark['Adj Close']

    # Calculating the monthly returns for tickers and benchmark
    df_tickers_monthly_returns = df_tickers.resample('M').ffill().pct_change()
    df_benchmark_monthly_returns = df_benchmark.resample('M').ffill().pct_change()
    df_tickers_monthly_returns_sum = (df_tickers_monthly_returns * wts).sum(axis = 1)

    # Calculating standard deviation for tickers and benchmark
    std_tickers_monthly_returns = df_tickers_monthly_returns.std()
    std_benchmark_monthly_returns = df_benchmark_monthly_returns.std()
    #print('Risk of each ticker:')
    #print(round(std_tickers_monthly_returns, 3))

    # Calculating standard deviation for portfolio
    cov_tickers = df_tickers_monthly_returns.cov()
    var_portfolio = np.dot(np.array(wts), np.dot(cov_tickers, np.array(wts)))
    std_portfolio = np.sqrt(var_portfolio)

    # Calculating the expected return on tickers, benchmark and portfolio
    exp_return_monthly = df_tickers_monthly_returns.mean()
    exp_return_monthly_benchmark = df_benchmark_monthly_returns.mean()
    exp_return_monthly_sum = (exp_return_monthly * wts).sum()
    #print('Return on each ticker:')
    #print(round(exp_return_monthly, 3))

    # Calculating the correlation coefficients between each ticker and benchmark
    corr_list = []
    for i in tickers:
        corr_ticker_to_benchmark = df_tickers_monthly_returns[i].corr(df_benchmark_monthly_returns)
        corr_list.append(corr_ticker_to_benchmark)
    corr_tickers_monthly_returns = pd.DataFrame(corr_list, index = tickers)[0]
    #print('Correlation between tickers and benchmark:')
    #print(round(corr_tickers_monthly_returns, 3))

    # Calculating the correlation between tickers
    corr_tabel_monthly = df_tickers_monthly_returns.corr()

    # Calculating the correlation between portfolio and benchmark
    corr_portfolio_monthly_returns = df_tickers_monthly_returns_sum.corr(df_benchmark_monthly_returns)

    # Calculating the beta value for tickers(5 year monthly)
    beta_list = []
    for i in tickers:
        beta_ticker = corr_tickers_monthly_returns[i] * (std_tickers_monthly_returns[i]/std_benchmark_monthly_returns)
        beta_list.append(beta_ticker)
    beta_tickers_monthly_returns = pd.DataFrame(beta_list, index = tickers)[0]
    #print('Beta values for tickers in portfolio:')
    #print(round(beta_tickers_monthly_returns, 2))

    # Calculating alpha value for tickers(5 year monthly)
    alpha_list = []
    for i in tickers:
        alpha_tickers = (exp_return_monthly[i] - beta_tickers_monthly_returns[i] * exp_return_monthly_benchmark)*100
        alpha_list.append(alpha_tickers)
    alpha_tickers_monthly_returns = pd.DataFrame(alpha_list, index = tickers)[0]
    #print('Alpha values for tickers in portfolio:')
    #print(round(alpha_tickers_monthly_returns, 2))

    # Calculating beta and alpha value for portfolio(5 year monthly)
    beta_portfolio_monthly_returns = (beta_tickers_monthly_returns * wts).sum()
    alpha_portfolio_monthly_returns = (alpha_tickers_monthly_returns * wts).sum()

    # Calculating the required return on the portfolio given the beta value
    req_return_monthly = RF_return + beta_portfolio_monthly_returns * (exp_return_monthly_benchmark - RF_return)

    # Combining ticker trading information into one dataframe
    table_tickers = pd.concat([std_tickers_monthly_returns,
                                exp_return_monthly,
                                corr_tickers_monthly_returns,
                                round(beta_tickers_monthly_returns, 2),
                                round(alpha_tickers_monthly_returns, 2)], axis=1)
    table_tickers.columns = ['Risk', 'Expected return', 'Correlation to benchmark', 'Beta', 'Alpha']
    print('Trading Information About Tickers:')
    print(table_tickers.round(3).T.astype(object))

    # Getting key values for tickers from financialmodelingprep
    ROE_list = []
    ROA_list = []
    ROIC_list = []
    DE_list = []
    DA_list = []
    Current_ratio_list = []
    PE_list = []
    finance_list = []
    for i in tickers:
        ticker_financial_data = requests.get(f'https://financialmodelingprep.com/api/v3/company-key-metrics/{i}')
        ticker_financial_data = ticker_financial_data.json()
        ROE = float(ticker_financial_data['metrics'][0]['ROE'])
        ROE_list.append(ROE)
        ROA = float(ticker_financial_data['metrics'][0]['Return on Tangible Assets'])
        ROA_list.append(ROA)
        ROIC = float(ticker_financial_data['metrics'][0]['ROIC'])
        ROIC_list.append(ROIC)
        DE = float(ticker_financial_data['metrics'][0]['Debt to Equity'])
        DE_list.append(DE)
        DA = float(ticker_financial_data['metrics'][0]['Debt to Assets'])
        DA_list.append(DA)
        Current_ratio = float(ticker_financial_data['metrics'][0]['Current ratio'])
        Current_ratio_list.append(Current_ratio)
        PE = float(ticker_financial_data['metrics'][0]['PE ratio'])
        PE_list.append(PE)

    table_tickers_finance = pd.DataFrame([ROE_list,
                                    ROA_list,
                                    ROIC_list,
                                    DE_list,
                                    DA_list,
                                    Current_ratio_list,
                                    PE_list], columns = tickers, index = ['Return on Equity',
                                                                'Return on Tangible Assets',
                                                                'Return on Invested Cap',
                                                                'Debt to Equity',
                                                                'Debt to Assets',
                                                                'Current Ratio',
                                                                'Price/Earnings Ratio'])
    print('')
    print('\nFinancial Highlights About Tickers:')
    print(table_tickers_finance.round(2))

    # Creating a dataframe for the portfolio trading information
    table_portfolio = pd.DataFrame([std_portfolio,
                                    exp_return_monthly_sum,
                                    req_return_monthly,
                                    corr_portfolio_monthly_returns,
                                    beta_portfolio_monthly_returns,
                                    alpha_portfolio_monthly_returns], columns = None,
                                                                    index = ['Risk',
                                                                            'Expected return',
                                                                            'Required return',
                                                                            'Correlation to benchmark',
                                                                            'Beta',
                                                                            'Alpha'])[0]
    print('')
    print('\nTrading Information About Portfolio:')
    print(table_portfolio.round(3))

    # Plotting heatmap for portfolio
    fig = plt.figure(figsize=(9, 6))
    heatmap = sns.heatmap(corr_tabel_monthly, annot = True, cmap='RdYlGn', vmin=-1, vmax=1)
    ax = fig.add_subplot(1, 1, 1)
    ax.xaxis.tick_top()

    # Plotting ticker prices
    tickers_plot = df_tickers.apply(lambda x: x / x[0])
    tickers_plot.head() - 1
    tickers_plot.plot(linewidth = 0.8, linestyle = '-')
    plt.grid(color = '#D3D3D3', linestyle = '--', linewidth = 0.5)
    plt.legend(loc='upper left', fontsize = 8)
    plt.yticks(fontsize = 8, fontname = font)
    plt.xticks(fontsize = 8, fontname = font)
    plt.show()

portfolio_analysis()
```
The result of this function will be the calculated values presented in the output tab as demonstrated beneath.
```
Trading Information About The Tickers:
                           TSLA     FB   AAPL   AMZN   GOOG
Risk                      0.141  0.072  0.078  0.083   0.06
Expected return           0.031  0.017  0.018  0.031  0.017
Correlation to benchmark  0.121  0.527  0.594  0.669  0.614
Beta                       0.48   1.05   1.29   1.55   1.02
Alpha                      2.75   0.99   0.92   2.08   1.01

Financial Highlights About The Tickers:
                            TSLA     FB   AAPL   AMZN   GOOG
Return on Equity           -0.20   0.18   0.61   0.19   0.17
Return on Tangible Assets  -0.03   0.16   0.16   0.06   0.14
Return on Invested Cap     -0.02   0.20   0.27   0.11   0.15
Debt to Equity              5.04   0.32   2.74   2.63   0.37
Debt to Assets              0.83   0.24   0.73   0.72   0.27
Current Ratio               0.83   4.40   1.54   1.10   3.37
Price/Earnings Ratio      -62.63  30.83  17.47  75.95  14.70

Trading Information About The Portfolio:
Risk                        0.061
Expected return             0.025
Required return             0.006
Correlation to benchmark    0.676
Beta                        1.151
Alpha                       1.774
```
The price evolution for the stocks in the portfolio:
![Portfolio_movment](https://user-images.githubusercontent.com/63104057/78459048-968a6880-76b6-11ea-8961-fa5eb5423239.png)
The correlation table for the stocks in the portfolio:
![Portfolio_correlation_table](https://user-images.githubusercontent.com/63104057/78459090-ed903d80-76b6-11ea-8753-9df50e2f6d10.png)

The last function does pretty much the same as the previous, but only for one ticker. 
```python
def ticker_analysis():
    df_ticker = web.DataReader(ticker, 'yahoo', start, end)
    df_ticker = df_ticker['Adj Close']
    df_benchmark = web.DataReader(benchmark, 'yahoo', start, end)
    df_benchmark = df_benchmark['Adj Close']

    # Calculating the monthly returns
    df_ticker_monthly_returns = df_ticker.resample('M').ffill().pct_change()
    df_benchmark_monthly_returns = df_benchmark.resample('M').ffill().pct_change()

    # Calculating the expected return on the ticker and benchmark
    exp_return_monthly = df_ticker_monthly_returns.mean()
    exp_return_monthly_benchmark = df_benchmark_monthly_returns.mean()

    # Calculating standard deviation for ticker and benchmark
    std_ticker_monthly_returns = df_ticker_monthly_returns.std()
    std_benchmark_monthly_returns = df_benchmark_monthly_returns.std()
    std_relative = std_ticker_monthly_returns/std_benchmark_monthly_returns
    #print(std_ticker_monthly_returns)

    # Calculating the correlation coefficient between ticker and benchmark
    corr_monthly = df_ticker_monthly_returns.corr(df_benchmark_monthly_returns)

    # Calculating the beta and alpha value for ticker (5 year monthly)
    beta = corr_monthly * (std_ticker_monthly_returns/std_benchmark_monthly_returns)
    alpha = (exp_return_monthly - beta * exp_return_monthly_benchmark)*100

    # Calculating the required return on the portfolio given the beta value
    req_return_monthly = RF_return + beta * (exp_return_monthly_benchmark - RF_return)

    # Creating a dataframe for the portfolio trading information
    table_ticker_trading = pd.DataFrame([std_ticker_monthly_returns,
                                    exp_return_monthly,
                                    req_return_monthly,
                                    corr_monthly,
                                    round(beta, 2),
                                    round(alpha, 2)], columns = None, index = ['Risk',
                                                                            'Expected return',
                                                                            'Required return',
                                                                            'Correlation to benchmark',
                                                                            'Beta',
                                                                            'Alpha'])[0]
    print('\nTrading Information About ', ticker, ':', sep='')
    print(table_ticker_trading.round(3).astype(object))

    # Getting key values for ticker from financialmodelingprep
    ticker_financial_data = requests.get(f'https://financialmodelingprep.com/api/v3/company-key-metrics/{ticker}')
    ticker_financial_data = ticker_financial_data.json()

    ROE = float(ticker_financial_data['metrics'][0]['ROE'])
    ROA = float(ticker_financial_data['metrics'][0]['Return on Tangible Assets'])
    ROIC = float(ticker_financial_data['metrics'][0]['ROIC'])
    DE = float(ticker_financial_data['metrics'][0]['Debt to Equity'])
    DA = float(ticker_financial_data['metrics'][0]['Debt to Assets'])
    Current_ratio = float(ticker_financial_data['metrics'][0]['Current ratio'])
    PE = float(ticker_financial_data['metrics'][0]['PE ratio'])

    table_ticker_finance = pd.DataFrame([ROE,
                                    ROA,
                                    ROIC,
                                    DE,
                                    DA,
                                    Current_ratio,
                                    PE], columns = None, index = ['Return on Equity',
                                                                'Return on Tangible Assets',
                                                                'Return on Invested Capital',
                                                                'Debt to Equity',
                                                                'Debt to Assets',
                                                                'Current Ratio',
                                                                'Price/Earnings Ratio'])[0]
    print('')
    print('\nFinancial Highlights About ', ticker, ':', sep='')
    print(table_ticker_finance.round(2))

    # Plotting ticker price
    plt.plot(df_ticker, color = '#188fff', linewidth = 0.8, linestyle = '-')
    plt.grid(color = '#D3D3D3', linestyle = '--', linewidth = 0.5)
    plt.yticks(fontsize = 8, fontname = font)
    plt.xticks(fontsize = 8, fontname = font)
    plt.show()

ticker_analysis()
```
The output of this function will be the calulatede values:
```
Trading Information About AAPL:
Risk                        0.078
Expected return             0.018
Required return             0.005
Correlation to benchmark    0.594
Beta                         1.29
Alpha                        0.92

Financial Highlights About AAPL:
Return on Equity               0.61
Return on Tangible Assets      0.16
Return on Invested Capital     0.27
Debt to Equity                 2.74
Debt to Assets                 0.73
Current Ratio                  1.54
Price/Earnings Ratio          17.47
```
And the figure showing the price evolution for the given ticker:
![Ticker_movment](https://user-images.githubusercontent.com/63104057/78459143-5081d480-76b7-11ea-9651-21de75839ba5.png)
