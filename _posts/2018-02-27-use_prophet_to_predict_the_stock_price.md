---
layout: post
title: 使用prophet预测股票价格
---
prophet介绍
--

* Prophet是Facebook于2017年2月开源的一款支持python和R的大规模时序预测工具
* 趋势预测+趋势分解
* 突变点识别+调整
* 异常值/离群值检测


使用prophet预测股票价格
--

```
import tushare as ts
import matplotlib.pyplot as plt
from fbprophet import Prophet

def GetData(start_date, end_date):
    raw_data = ts.get_k_data('000300', index=True,
                             start=start_date, end=end_date)
    selected_data = raw_data.loc[:, ['date', 'close']]
    return selected_data

def main():
    data_df = GetData("2012-03", "2017-12")
    data_df["ds"] = data_df["date"]
    data_df["y"] = data_df["close"]

    print(data_df.head())
    print(data_df.tail())

    m = Prophet()
    m.fit(data_df)

    # make prediction
    data_future = m.make_future_dataframe(periods=120, freq="D")
    print(data_future.tail())

    pred_res = m.predict(data_future)
    print(pred_res[['ds', 'yhat', 'yhat_lower', 'yhat_upper']].tail())

    m.plot(pred_res)

    m.plot_components(pred_res)

    plt.show()

if __name__ == '__main__':
    main()
```