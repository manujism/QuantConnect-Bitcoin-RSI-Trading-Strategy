# QuantConnect-Bitcoin-RSI-Trading-Strategy

from AlgorithmImports import *

class RSICrossStrategy(QCAlgorithm):
    def Initialize(self):
        self.SetStartDate(2020, 1, 1)
        self.SetEndDate(2023, 1, 1)
        self.SetCash(100000)
        
        self.SetBrokerageModel(BrokerageName.Bitfinex)
        self.symbol = self.AddCrypto("BTCUSD", Resolution.Daily).Symbol
        self.rsi = self.RSI(self.symbol, 14)
        
        self.previous_rsi = None
        self.two_ago_rsi = None
        self.entry_ticket = None
        self.entry_price = None
        self.stop_loss_percent = 0.15  # 15% stop loss

    def OnData(self, data):
        if not self.rsi.IsReady or self.symbol not in data or not data[self.symbol]:
            return
        
        current_rsi = self.rsi.Current.Value
        price = data[self.symbol].Close

        # Check for stop loss if in position
        if self.Portfolio[self.symbol].Invested and self.entry_price is not None:
            stop_price = self.entry_price * (1 - self.stop_loss_percent)
            if price <= stop_price:
                self.Liquidate(self.symbol, "Stop loss triggered")
                self.entry_price = None  # Reset entry price
                return

        if self.two_ago_rsi is not None and self.previous_rsi is not None:
            # Entry condition: RSI crosses above 40
            if self.previous_rsi > 40 and self.two_ago_rsi <= 40:
                if not self.Portfolio[self.symbol].Invested:
                    quantity = self.CalculateOrderQuantity(self.symbol, 1)
                    ticket = self.MarketOrder(self.symbol, quantity)
                    self.entry_price = price  # Store entry price for stop loss

            # Exit condition: RSI crosses below 40
            elif self.previous_rsi < 40 and self.two_ago_rsi >= 40:
                if self.Portfolio[self.symbol].Invested:
                    self.Liquidate(self.symbol, "RSI exit")
                    self.entry_price = None  # Reset entry price

        # Update historical RSI values
        self.two_ago_rsi = self.previous_rsi
        self.previous_rsi = current_rsi

