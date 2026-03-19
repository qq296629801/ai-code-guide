# 爆品预测算法 - 技术调研报告

> 版本：v1.0  
> 日期：2026-03-19  
> 作者：AI Assistant

---

## 一、调研概述

### 1.1 调研目标

本报告旨在调研电商爆品预测领域的最新算法技术，包括：
- 时序预测算法
- 机器学习方法
- 深度学习模型
- 多模态融合技术
- 工业实践案例

### 1.2 问题定义

**爆品预测问题**：基于商品的历史销量、价格、评价、搜索热度、季节因素等多维度数据，预测商品在未来一段时间内是否会成为爆款。

**数学定义**：
```
给定商品 i 在时间 t 的特征向量 X_i(t)，预测未来 k 天的销量或爆品概率：

销量预测：ŷ_i(t+k) = f(X_i(t), X_i(t-1), ..., X_i(t-n))
爆品分类：P(hot_i) = g(X_i(t), context)

其中爆品定义为：销量在类目中排名前 10%
```

### 1.3 应用场景

| 场景 | 预测目标 | 预测周期 |
|------|----------|----------|
| 选品决策 | 爆品概率 | 未来 7-30 天 |
| 库存管理 | 销量预测 | 未来 3-7 天 |
| 定价策略 | 最优价格 | 实时 |
| 营销投放 | ROI 预测 | 未来 1-3 天 |

---

## 二、算法分类与对比

### 2.1 算法体系

```
爆品预测算法
├── 统计方法
│   ├── 移动平均 (MA)
│   ├── 指数平滑 (ETS)
│   ├── ARIMA
│   └── Prophet
│
├── 机器学习
│   ├── 线性回归 / Ridge / Lasso
│   ├── 决策树 (CART)
│   ├── 集成方法
│   │   ├── Random Forest
│   │   ├── XGBoost
│   │   ├── LightGBM
│   │   └── CatBoost
│   └── SVM
│
├── 深度学习
│   ├── RNN / LSTM / GRU
│   ├── TCN (时序卷积)
│   ├── Transformer
│   │   ├── Informer
│   │   ├── Autoformer
│   │   └── FEDformer
│   └── N-BEATS
│
├── 多模态融合
│   ├── 图像 + 文本
│   ├── 图像 + 数值
│   └── 文本 + 数值
│
└── 混合模型
    ├── 统计 + ML
    ├── ML + DL
    └── 多模型集成
```

### 2.2 算法对比

| 算法 | 准确率 | 可解释性 | 训练速度 | 推理速度 | 数据需求 | 适用场景 |
|------|--------|----------|----------|----------|----------|----------|
| ARIMA | 中 | 高 | 快 | 快 | 低 | 短期预测 |
| Prophet | 中 | 高 | 快 | 快 | 低 | 季节性预测 |
| XGBoost | 高 | 中 | 中 | 快 | 中 | 特征丰富 |
| LightGBM | 高 | 中 | 快 | 快 | 中 | 大规模数据 |
| LSTM | 高 | 低 | 慢 | 中 | 高 | 长序列 |
| Transformer | 最高 | 低 | 慢 | 中 | 高 | 复杂模式 |
| 多模态 | 最高 | 低 | 慢 | 慢 | 高 | 图文商品 |

---

## 三、时序预测算法详解

### 3.1 传统统计方法

#### 3.1.1 ARIMA (自回归积分滑动平均模型)

**原理**：
```
ARIMA(p, d, q) 由三部分组成：
- AR(p): 自回归项，用过去 p 个值预测当前值
- I(d): 差分次数，使序列平稳
- MA(q): 滑动平均项，用过去 q 个误差项修正

模型形式：
y_t = c + φ_1*y_{t-1} + ... + φ_p*y_{t-p} + θ_1*ε_{t-1} + ... + θ_q*ε_{t-q} + ε_t
```

**优点**：
- 理论成熟，可解释性强
- 对线性趋势效果好
- 数据需求少

**缺点**：
- 难以处理非线性关系
- 需要序列平稳
- 无法利用外部特征

**Python 实现**：
```python
from statsmodels.tsa.arima.model import ARIMA
import pandas as pd

# 销量数据
sales = pd.Series([100, 120, 115, 130, 125, 140, 150])

# 拟合 ARIMA 模型
model = ARIMA(sales, order=(2, 1, 2))
fitted = model.fit()

# 预测未来 7 天
forecast = fitted.forecast(steps=7)
print(forecast)
```

**适用场景**：
- 单商品销量预测
- 数据量较少时
- 作为基线模型

---

#### 3.1.2 Prophet (Facebook)

**原理**：
```
Prophet 将时间序列分解为：
y(t) = g(t) + s(t) + h(t) + ε_t

其中：
- g(t): 趋势项（分段线性或逻辑增长）
- s(t): 季节项（傅里叶级数）
- h(t): 节假日效应
- ε_t: 误差项
```

**优点**：
- 自动处理季节性和节假日
- 对异常值鲁棒
- 不需要序列平稳
- 参数易调

**Python 实现**：
```python
from prophet import Prophet
import pandas as pd

# 数据格式要求
df = pd.DataFrame({
    'ds': pd.date_range('2025-01-01', periods=365),
    'y': sales_data
})

# 创建模型
model = Prophet(
    yearly_seasonality=True,
    weekly_seasonality=True,
    daily_seasonality=False,
    holidays=holidays_df
)

# 添加自定义季节性
model.add_seasonality(name='monthly', period=30.5, fourier_order=5)

# 拟合
model.fit(df)

# 预测
future = model.make_future_dataframe(periods=30)
forecast = model.predict(future)

# 可视化
fig = model.plot(forecast)
fig2 = model.plot_components(forecast)
```

**适用场景**：
- 有明显季节性的商品（服装、节日礼品）
- 快速原型开发
- 业务分析师使用

---

### 3.2 机器学习方法

#### 3.2.1 XGBoost

**原理**：
```
XGBoost 是 Gradient Boosting Decision Tree 的优化实现：

目标函数：
Obj = Σ L(y_i, ŷ_i) + Σ Ω(f_k)

其中：
- L: 损失函数（MSE, LogLoss 等）
- Ω: 正则化项，控制模型复杂度

优化策略：
- 二阶泰勒展开
- 列采样
- 行采样
- 稀疏特征处理
```

**优点**：
- 预测准确率高
- 自动处理缺失值
- 特征重要性可解释
- 支持并行计算

**销量预测实现**：
```python
import xgboost as xgb
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, mean_absolute_percentage_error
import numpy as np

class SalesPredictor:
    def __init__(self):
        self.model = None
        self.feature_names = None
        
    def prepare_features(self, df):
        """准备特征"""
        features = []
        
        # 滞后特征
        for lag in [1, 3, 7, 14, 30]:
            features.append(f'sales_lag_{lag}')
            df[f'sales_lag_{lag}'] = df.groupby('product_id')['sales'].shift(lag)
        
        # 滚动统计特征
        for window in [7, 14, 30]:
            df[f'sales_rolling_mean_{window}'] = df.groupby('product_id')['sales'].transform(
                lambda x: x.rolling(window, min_periods=1).mean()
            )
            df[f'sales_rolling_std_{window}'] = df.groupby('product_id')['sales'].transform(
                lambda x: x.rolling(window, min_periods=1).std()
            )
            features.extend([f'sales_rolling_mean_{window}', f'sales_rolling_std_{window}'])
        
        # 时间特征
        df['day_of_week'] = df['date'].dt.dayofweek
        df['day_of_month'] = df['date'].dt.day
        df['month'] = df['date'].dt.month
        df['is_weekend'] = (df['day_of_week'] >= 5).astype(int)
        features.extend(['day_of_week', 'day_of_month', 'month', 'is_weekend'])
        
        # 价格特征
        features.extend(['price', 'discount_rate'])
        
        # 类目特征
        features.append('category_id')
        
        return df, features
    
    def train(self, df, target_col='sales'):
        """训练模型"""
        df, features = self.prepare_features(df)
        self.feature_names = features
        
        # 划分数据
        X = df[features].fillna(0)
        y = df[target_col]
        
        X_train, X_val, y_train, y_val = train_test_split(
            X, y, test_size=0.2, random_state=42
        )
        
        # 训练
        self.model = xgb.XGBRegressor(
            n_estimators=500,
            max_depth=8,
            learning_rate=0.05,
            subsample=0.8,
            colsample_bytree=0.8,
            random_state=42,
            n_jobs=-1
        )
        
        self.model.fit(
            X_train, y_train,
            eval_set=[(X_val, y_val)],
            early_stopping_rounds=50,
            verbose=10
        )
        
        # 评估
        y_pred = self.model.predict(X_val)
        rmse = np.sqrt(mean_squared_error(y_val, y_pred))
        mape = mean_absolute_percentage_error(y_val, y_pred)
        
        print(f"RMSE: {rmse:.2f}")
        print(f"MAPE: {mape:.2%}")
        
        return self
    
    def predict(self, df):
        """预测"""
        df, _ = self.prepare_features(df)
        X = df[self.feature_names].fillna(0)
        return self.model.predict(X)
    
    def get_feature_importance(self, top_n=20):
        """特征重要性"""
        importance = pd.DataFrame({
            'feature': self.feature_names,
            'importance': self.model.feature_importances_
        }).sort_values('importance', ascending=False)
        
        return importance.head(top_n)
```

**爆品分类实现**：
```python
class HotProductClassifier:
    """爆品分类器"""
    
    def __init__(self):
        self.model = None
        
    def train(self, X, y):
        """训练"""
        # 处理类别不平衡
        scale_pos_weight = len(y[y==0]) / len(y[y==1])
        
        self.model = xgb.XGBClassifier(
            objective='binary:logistic',
            eval_metric='auc',
            n_estimators=500,
            max_depth=8,
            learning_rate=0.05,
            subsample=0.8,
            colsample_bytree=0.8,
            scale_pos_weight=scale_pos_weight,
            random_state=42,
            n_jobs=-1
        )
        
        X_train, X_val, y_train, y_val = train_test_split(
            X, y, test_size=0.2, random_state=42, stratify=y
        )
        
        self.model.fit(
            X_train, y_train,
            eval_set=[(X_val, y_val)],
            early_stopping_rounds=50
        )
        
        # 评估
        from sklearn.metrics import classification_report, roc_auc_score
        
        y_pred = self.model.predict(X_val)
        y_proba = self.model.predict_proba(X_val)[:, 1]
        
        print(classification_report(y_val, y_pred))
        print(f"AUC: {roc_auc_score(y_val, y_proba):.4f}")
        
        return self
```

---

#### 3.2.2 LightGBM

**原理**：
```
LightGBM 相比 XGBoost 的优化：

1. 直方图算法
   - 将连续值离散化为 k 个 bins
   - 加速分裂点查找

2. Leaf-wise 生长策略
   - 只分裂增益最大的叶子
   - 更快收敛，但可能过拟合

3. GOSS (梯度单边采样)
   - 保留大梯度样本
   - 随机采样小梯度样本

4. EFB (互斥特征捆绑)
   - 稀疏特征合并
   - 减少特征维度
```

**性能对比**：
```
              训练速度    内存占用    预测精度
XGBoost        中         高         高
LightGBM       快         低         高
CatBoost       慢         高         高
```

**实现**：
```python
import lightgbm as lgb

class LightGBMPredictor:
    def __init__(self):
        self.model = None
        
    def train(self, X, y, categorical_features=None):
        """训练"""
        params = {
            'objective': 'regression',
            'metric': 'rmse',
            'boosting_type': 'gbdt',
            'num_leaves': 31,
            'learning_rate': 0.05,
            'feature_fraction': 0.8,
            'bagging_fraction': 0.8,
            'bagging_freq': 5,
            'verbose': -1,
            'n_jobs': -1
        }
        
        X_train, X_val, y_train, y_val = train_test_split(
            X, y, test_size=0.2, random_state=42
        )
        
        train_data = lgb.Dataset(X_train, label=y_train, 
                                  categorical_feature=categorical_features)
        val_data = lgb.Dataset(X_val, label=y_val, reference=train_data)
        
        self.model = lgb.train(
            params,
            train_data,
            num_boost_round=1000,
            valid_sets=[val_data],
            callbacks=[
                lgb.early_stopping(stopping_rounds=50),
                lgb.log_evaluation(period=100)
            ]
        )
        
        return self
    
    def predict(self, X):
        """预测"""
        return self.model.predict(X)
```

**适用场景**：
- 大规模商品预测
- 需要快速迭代
- 特征维度高

---

### 3.3 深度学习方法

#### 3.3.1 LSTM (长短期记忆网络)

**原理**：
```
LSTM 通过门控机制解决 RNN 的梯度消失问题：

遗忘门：f_t = σ(W_f·[h_{t-1}, x_t] + b_f)
输入门：i_t = σ(W_i·[h_{t-1}, x_t] + b_i)
候选值：c̃_t = tanh(W_c·[h_{t-1}, x_t] + b_c)
细胞状态：c_t = f_t * c_{t-1} + i_t * c̃_t
输出门：o_t = σ(W_o·[h_{t-1}, x_t] + b_o)
隐藏状态：h_t = o_t * tanh(c_t)

关键思想：
- 遗忘门：决定丢弃哪些信息
- 输入门：决定存储哪些新信息
- 输出门：决定输出哪些信息
```

**销量预测模型**：
```python
import torch
import torch.nn as nn
from torch.utils.data import Dataset, DataLoader
import numpy as np

class SalesDataset(Dataset):
    """销量数据集"""
    
    def __init__(self, data, seq_len=30, pred_len=7):
        self.data = torch.FloatTensor(data)
        self.seq_len = seq_len
        self.pred_len = pred_len
        
    def __len__(self):
        return len(self.data) - self.seq_len - self.pred_len
    
    def __getitem__(self, idx):
        x = self.data[idx:idx+self.seq_len]
        y = self.data[idx+self.seq_len:idx+self.seq_len+self.pred_len, 0]  # 预测销量
        return x, y


class LSTMSalesPredictor(nn.Module):
    """LSTM 销量预测模型"""
    
    def __init__(self, 
                 input_size, 
                 hidden_size=128, 
                 num_layers=2,
                 pred_len=7,
                 dropout=0.2):
        super().__init__()
        
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.pred_len = pred_len
        
        # LSTM 层
        self.lstm = nn.LSTM(
            input_size=input_size,
            hidden_size=hidden_size,
            num_layers=num_layers,
            batch_first=True,
            dropout=dropout if num_layers > 1 else 0
        )
        
        # 全连接层
        self.fc = nn.Sequential(
            nn.Linear(hidden_size, hidden_size // 2),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(hidden_size // 2, pred_len)
        )
        
    def forward(self, x):
        # x: (batch, seq_len, input_size)
        
        # LSTM
        lstm_out, (hidden, cell) = self.lstm(x)
        
        # 取最后一个时间步
        last_hidden = lstm_out[:, -1, :]
        
        # 预测
        output = self.fc(last_hidden)
        
        return output


def train_lstm_model(model, train_loader, val_loader, epochs=100, lr=0.001, device='cuda'):
    """训练 LSTM 模型"""
    model = model.to(device)
    
    criterion = nn.MSELoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=lr)
    scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
        optimizer, mode='min', factor=0.5, patience=5
    )
    
    best_val_loss = float('inf')
    patience_counter = 0
    
    for epoch in range(epochs):
        # 训练
        model.train()
        train_loss = 0
        
        for x, y in train_loader:
            x, y = x.to(device), y.to(device)
            
            optimizer.zero_grad()
            pred = model(x)
            loss = criterion(pred, y)
            loss.backward()
            
            # 梯度裁剪
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            
            optimizer.step()
            train_loss += loss.item()
        
        # 验证
        model.eval()
        val_loss = 0
        
        with torch.no_grad():
            for x, y in val_loader:
                x, y = x.to(device), y.to(device)
                pred = model(x)
                val_loss += criterion(pred, y).item()
        
        train_loss /= len(train_loader)
        val_loss /= len(val_loader)
        
        print(f"Epoch {epoch+1}/{epochs} - Train Loss: {train_loss:.4f}, Val Loss: {val_loss:.4f}")
        
        # 学习率调度
        scheduler.step(val_loss)
        
        # 早停
        if val_loss < best_val_loss:
            best_val_loss = val_loss
            torch.save(model.state_dict(), 'best_lstm_model.pth')
            patience_counter = 0
        else:
            patience_counter += 1
            if patience_counter >= 15:
                print("Early stopping!")
                break
    
    # 加载最佳模型
    model.load_state_dict(torch.load('best_lstm_model.pth'))
    return model
```

**适用场景**：
- 长序列预测（30+ 天历史）
- 复杂非线性关系
- GPU 资源充足

---

#### 3.3.2 Transformer (Time Series Transformer)

**原理**：
```
Transformer 应用于时序预测的核心组件：

1. 位置编码
   PE(pos, 2i) = sin(pos / 10000^(2i/d_model))
   PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))

2. 自注意力
   Attention(Q, K, V) = softmax(QK^T / √d_k) V

3. 多头注意力
   MultiHead(Q, K, V) = Concat(head_1, ..., head_h) W^O
   其中 head_i = Attention(QW_i^Q, KW_i^K, VW_i^V)

4. 编码器-解码器结构
   - Encoder: 提取历史序列特征
   - Decoder: 自回归生成预测
```

**优势**：
- 并行计算，训练速度快
- 能够捕捉长距离依赖
- 可解释性好（注意力权重）

**Informer (高效 Transformer)**：
```python
"""
Informer: 超长序列预测的高效 Transformer

核心改进：
1. ProbSparse Self-Attention
   - 只计算 Top-u 个关键 query
   - 复杂度从 O(n²) 降到 O(n log n)

2. Self-Attention Distilling
   - 逐层减少序列长度
   - 提取关键特征

3. Generative Decoder
   - 一次性生成整个预测序列
   - 避免逐步解码的速度瓶颈
"""

import torch
import torch.nn as nn
import math

class ProbSparseAttention(nn.Module):
    """ProbSparse 自注意力"""
    
    def __init__(self, d_model, n_heads, sampling_factor=5):
        super().__init__()
        self.d_model = d_model
        self.n_heads = n_heads
        self.sampling_factor = sampling_factor
        self.head_dim = d_model // n_heads
        
        self.W_Q = nn.Linear(d_model, d_model)
        self.W_K = nn.Linear(d_model, d_model)
        self.W_V = nn.Linear(d_model, d_model)
        self.W_O = nn.Linear(d_model, d_model)
        
    def _prob_QK(self, Q, K, sample_k, n_top):
        """采样计算稀疏注意力"""
        B, H, L_K, E = K.shape
        _, _, L_Q, _ = Q.shape
        
        # 随机采样 K
        K_sample = K[:, :, torch.randperm(L_K)[:sample_k], :]
        
        # 计算 Q-K 相似度
        Q_K_sample = torch.matmul(Q, K_sample.transpose(-2, -1))  # (B, H, L_Q, sample_k)
        
        # 找到 Top-u 个重要的 Q
        M = Q_K_sample.max(dim=-1)[0] - Q_K_sample.sum(dim=-1) / L_K
        M_top = M.topk(n_top, dim=-1)[1]
        
        # 只计算这些 Q 的注意力
        Q_reduce = torch.gather(Q, 2, M_top.unsqueeze(-1).expand(-1, -1, -1, E))
        Q_K = torch.matmul(Q_reduce, K.transpose(-2, -1))
        
        return Q_K, M_top
    
    def forward(self, Q, K, V):
        B, L_Q, _ = Q.shape
        _, L_K, _ = K.shape
        
        # 线性变换
        Q = self.W_Q(Q).view(B, L_Q, self.n_heads, self.head_dim).transpose(1, 2)
        K = self.W_K(K).view(B, L_K, self.n_heads, self.head_dim).transpose(1, 2)
        V = self.W_V(V).view(B, L_K, self.n_heads, self.head_dim).transpose(1, 2)
        
        # 稀疏采样
        sample_k = int(self.sampling_factor * math.log(L_K))
        n_top = int(self.sampling_factor * math.log(L_Q))
        
        Q_K, M_top = self._prob_QK(Q, K, sample_k, n_top)
        
        # 计算注意力
        attn = torch.softmax(Q_K / math.sqrt(self.head_dim), dim=-1)
        
        # 应用到 V
        V_reduce = torch.matmul(attn, V)
        
        # 恢复完整序列
        output = torch.zeros_like(Q)
        output.scatter_(2, M_top.unsqueeze(-1).expand(-1, -1, -1, self.head_dim), V_reduce)
        
        # 合并多头
        output = output.transpose(1, 2).contiguous().view(B, L_Q, self.d_model)
        output = self.W_O(output)
        
        return output


class Informer(nn.Module):
    """Informer 时序预测模型"""
    
    def __init__(self, 
                 enc_in,    # 编码器输入特征数
                 dec_in,    # 解码器输入特征数
                 c_out,     # 输出特征数
                 seq_len,   # 输入序列长度
                 label_len, # 标签长度（解码器输入）
                 pred_len,  # 预测长度
                 d_model=512,
                 n_heads=8,
                 e_layers=3,
                 d_layers=2,
                 d_ff=2048,
                 dropout=0.1):
        super().__init__()
        
        self.seq_len = seq_len
        self.label_len = label_len
        self.pred_len = pred_len
        
        # 编码器
        self.enc_embedding = nn.Linear(enc_in, d_model)
        self.pos_embedding = PositionalEncoding(d_model, dropout, max_len=seq_len)
        
        encoder_layers = nn.ModuleList([
            EncoderLayer(d_model, n_heads, d_ff, dropout)
            for _ in range(e_layers)
        ])
        self.encoder = nn.ModuleList(encoder_layers)
        
        # 解码器
        self.dec_embedding = nn.Linear(dec_in, d_model)
        self.dec_pos_embedding = PositionalEncoding(d_model, dropout, max_len=label_len + pred_len)
        
        decoder_layers = nn.ModuleList([
            DecoderLayer(d_model, n_heads, d_ff, dropout)
            for _ in range(d_layers)
        ])
        self.decoder = nn.ModuleList(decoder_layers)
        
        # 输出层
        self.projection = nn.Linear(d_model, c_out)
        
    def forward(self, x_enc, x_dec):
        # 编码器
        enc_out = self.enc_embedding(x_enc)
        enc_out = self.pos_embedding(enc_out)
        
        for layer in self.encoder:
            enc_out = layer(enc_out)
        
        # 解码器
        dec_out = self.dec_embedding(x_dec)
        dec_out = self.dec_pos_embedding(dec_out)
        
        for layer in self.decoder:
            dec_out = layer(dec_out, enc_out)
        
        # 输出
        dec_out = self.projection(dec_out)
        
        return dec_out[:, -self.pred_len:, :]


class PositionalEncoding(nn.Module):
    """位置编码"""
    
    def __init__(self, d_model, dropout=0.1, max_len=5000):
        super().__init__()
        self.dropout = nn.Dropout(p=dropout)
        
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len, dtype=torch.float).unsqueeze(1)
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model))
        
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        
        pe = pe.unsqueeze(0)
        self.register_buffer('pe', pe)
        
    def forward(self, x):
        x = x + self.pe[:, :x.size(1), :]
        return self.dropout(x)
```

**适用场景**：
- 超长序列预测（96+ 步）
- 多变量预测
- 需要并行训练

---

#### 3.3.3 N-BEATS (Neural Basis Expansion)

**原理**：
```
N-BEATS 使用全连接网络学习时序基函数：

1. 双残差结构
   - 前向残差：x_l = x_{l-1} - backcast_l
   - 后向堆叠：forecast = Σ forecast_l

2. 基函数展开
   backcast_l, forecast_l = FC_stack(theta_l)
   
   其中 theta_l 是基函数系数

3. 解释性变体
   - 趋势基函数：T(t) = Σ a_i * t^i
   - 季节基函数：S(t) = Σ b_i * sin(2πi*t/T) + c_i * cos(2πi*t/T)
```

**实现**：
```python
class NBeatsBlock(nn.Module):
    """N-BEATS 块"""
    
    def __init__(self, input_len, output_len, hidden_size, n_layers=4):
        super().__init__()
        
        self.input_len = input_len
        self.output_len = output_len
        
        # 全连接层
        layers = []
        for i in range(n_layers):
            layers.append(nn.Linear(input_len if i == 0 else hidden_size, hidden_size))
            layers.append(nn.ReLU())
        self.fc = nn.Sequential(*layers)
        
        # 基函数系数
        self.theta_backcast = nn.Linear(hidden_size, input_len)
        self.theta_forecast = nn.Linear(hidden_size, output_len)
        
    def forward(self, x):
        # 全连接
        h = self.fc(x)
        
        # 基函数展开
        backcast = self.theta_backcast(h)
        forecast = self.theta_forecast(h)
        
        return backcast, forecast


class NBeats(nn.Module):
    """N-BEATS 模型"""
    
    def __init__(self, input_len, output_len, hidden_size=256, n_blocks=30, n_layers=4):
        super().__init__()
        
        self.blocks = nn.ModuleList([
            NBeatsBlock(input_len, output_len, hidden_size, n_layers)
            for _ in range(n_blocks)
        ])
        
    def forward(self, x):
        forecast = torch.zeros(x.size(0), self.blocks[0].output_len, device=x.device)
        
        for block in self.blocks:
            backcast, forecast_block = block(x)
            x = x - backcast  # 残差
            forecast = forecast + forecast_block  # 累加预测
        
        return forecast
```

**适用场景**：
- 快速建模
- 需要可解释性
- 单变量预测

---

## 四、多模态融合

### 4.1 商品图文融合

**架构**：
```
商品图片 ──→ Vision Encoder (ViT/ResNet) ──→ 图像嵌入 (768d)
                                               │
                                               ├── 融合层 ──→ 预测
                                               │
商品标题/描述 ──→ Text Encoder (BERT) ──→ 文本嵌入 (768d)

数值特征 ──→ MLP ──→ 数值嵌入 (256d) ─────────┘
```

**实现**：
```python
import torch
import torch.nn as nn
from transformers import BertModel, ViTModel

class MultiModalProductPredictor(nn.Module):
    """多模态商品预测模型"""
    
    def __init__(self,
                 text_model='bert-base-chinese',
                 image_model='google/vit-base-patch16-224',
                 num_features=50,
                 hidden_size=256,
                 output_size=1):
        super().__init__()
        
        # 文本编码器
        self.text_encoder = BertModel.from_pretrained(text_model)
        text_dim = self.text_encoder.config.hidden_size
        
        # 图像编码器
        self.image_encoder = ViTModel.from_pretrained(image_model)
        image_dim = self.image_encoder.config.hidden_size
        
        # 数值特征编码
        self.num_encoder = nn.Sequential(
            nn.Linear(num_features, hidden_size),
            nn.ReLU(),
            nn.Dropout(0.2),
            nn.Linear(hidden_size, hidden_size)
        )
        
        # 融合层
        fusion_dim = text_dim + image_dim + hidden_size
        self.fusion = nn.Sequential(
            nn.Linear(fusion_dim, hidden_size * 2),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(hidden_size * 2, hidden_size),
            nn.ReLU(),
            nn.Dropout(0.2)
        )
        
        # 输出层
        self.output = nn.Linear(hidden_size, output_size)
        
    def forward(self, text_input, image_input, num_input):
        # 文本编码
        text_out = self.text_encoder(**text_input)
        text_embed = text_out.last_hidden_state[:, 0, :]  # [CLS]
        
        # 图像编码
        image_out = self.image_encoder(**image_input)
        image_embed = image_out.last_hidden_state[:, 0, :]  # [CLS]
        
        # 数值编码
        num_embed = self.num_encoder(num_input)
        
        # 融合
        fused = torch.cat([text_embed, image_embed, num_embed], dim=-1)
        fused = self.fusion(fused)
        
        # 输出
        output = self.output(fused)
        
        return output
```

### 4.2 跨模态预训练

**CLIP 方式**：
```python
"""
使用 CLIP 进行商品图文表示学习：

1. 对比学习
   - 图文配对
   - 最大化配对相似度

2. 零样本预测
   - 无需微调即可预测
"""

from transformers import CLIPModel, CLIPProcessor

class CLIPProductEncoder:
    def __init__(self, model_name='openai/clip-vit-base-patch32'):
        self.model = CLIPModel.from_pretrained(model_name)
        self.processor = CLIPProcessor.from_pretrained(model_name)
        
    def encode(self, images, texts):
        """编码商品图文"""
        inputs = self.processor(
            text=texts,
            images=images,
            return_tensors='pt',
            padding=True,
            truncation=True
        )
        
        outputs = self.model(**inputs)
        
        return outputs.image_embeds, outputs.text_embeds
    
    def similarity(self, image_embed, text_embed):
        """计算图文相似度"""
        return torch.cosine_similarity(image_embed, text_embed)
```

---

## 五、特征工程

### 5.1 特征维度

#### 5.1.1 销量特征

```python
def create_sales_features(df):
    """销量特征工程"""
    features = []
    
    # 滞后特征
    for lag in [1, 3, 7, 14, 30]:
        df[f'sales_lag_{lag}'] = df.groupby('product_id')['sales'].shift(lag)
        features.append(f'sales_lag_{lag}')
    
    # 滚动统计
    for window in [7, 14, 30]:
        df[f'sales_mean_{window}'] = df.groupby('product_id')['sales'].transform(
            lambda x: x.rolling(window, min_periods=1).mean()
        )
        df[f'sales_std_{window}'] = df.groupby('product_id')['sales'].transform(
            lambda x: x.rolling(window, min_periods=1).std()
        )
        df[f'sales_max_{window}'] = df.groupby('product_id')['sales'].transform(
            lambda x: x.rolling(window, min_periods=1).max()
        )
        features.extend([f'sales_mean_{window}', f'sales_std_{window}', f'sales_max_{window}'])
    
    # 增长率
    df['sales_growth_1d'] = df.groupby('product_id')['sales'].pct_change()
    df['sales_growth_7d'] = df.groupby('product_id')['sales'].pct_change(7)
    features.extend(['sales_growth_1d', 'sales_growth_7d'])
    
    # 加速度
    df['sales_acceleration'] = df['sales_growth_1d'] - df.groupby('product_id')['sales_growth_1d'].shift(1)
    features.append('sales_acceleration')
    
    return df, features
```

#### 5.1.2 价格特征

```python
def create_price_features(df):
    """价格特征工程"""
    features = []
    
    # 折扣率
    df['discount_rate'] = (df['original_price'] - df['price']) / df['original_price']
    features.append('discount_rate')
    
    # 价格变化
    df['price_change_1d'] = df.groupby('product_id')['price'].diff()
    df['price_change_7d'] = df.groupby('product_id')['price'].diff(7)
    features.extend(['price_change_1d', 'price_change_7d'])
    
    # 价格分位数（同类商品）
    df['price_percentile'] = df.groupby(['category_id', 'date'])['price'].transform(
        lambda x: x.rank(pct=True)
    )
    features.append('price_percentile')
    
    # 价格波动
    df['price_volatility'] = df.groupby('product_id')['price'].transform(
        lambda x: x.rolling(7, min_periods=1).std() / x.rolling(7, min_periods=1).mean()
    )
    features.append('price_volatility')
    
    return df, features
```

#### 5.1.3 时间特征

```python
def create_time_features(df):
    """时间特征工程"""
    features = []
    
    # 基础时间
    df['day_of_week'] = df['date'].dt.dayofweek
    df['day_of_month'] = df['date'].dt.day
    df['month'] = df['date'].dt.month
    df['quarter'] = df['date'].dt.quarter
    df['is_weekend'] = (df['day_of_week'] >= 5).astype(int)
    features.extend(['day_of_week', 'day_of_month', 'month', 'quarter', 'is_weekend'])
    
    # 季节
    df['season'] = df['month'].apply(lambda x: (x % 12 + 3) // 3)
    features.append('season')
    
    # 节假日（需要节假日数据）
    df['is_holiday'] = df['date'].isin(holidays).astype(int)
    df['days_to_holiday'] = df['date'].apply(lambda x: min((h - x).days for h in holidays if h >= x))
    features.extend(['is_holiday', 'days_to_holiday'])
    
    # 周期性编码（正弦/余弦）
    df['day_sin'] = np.sin(2 * np.pi * df['day_of_week'] / 7)
    df['day_cos'] = np.cos(2 * np.pi * df['day_of_week'] / 7)
    df['month_sin'] = np.sin(2 * np.pi * df['month'] / 12)
    df['month_cos'] = np.cos(2 * np.pi * df['month'] / 12)
    features.extend(['day_sin', 'day_cos', 'month_sin', 'month_cos'])
    
    return df, features
```

#### 5.1.4 商品特征

```python
def create_product_features(df):
    """商品特征工程"""
    features = []
    
    # 基础属性
    features.extend(['category_id', 'brand_id'])
    
    # 上架时长
    df['days_since_launch'] = (df['date'] - df['launch_date']).dt.days
    features.append('days_since_launch')
    
    # 生命周期阶段
    df['lifecycle_stage'] = pd.cut(
        df['days_since_launch'],
        bins=[0, 7, 30, 90, 365, float('inf')],
        labels=['新品', '成长期', '成熟期', '稳定期', '老品']
    )
    features.append('lifecycle_stage')
    
    # 佣金率
    df['commission_rate'] = df['commission'] / df['price']
    features.append('commission_rate')
    
    return df, features
```

### 5.2 特征选择

```python
from sklearn.feature_selection import SelectKBest, mutual_info_regression, RFE
from sklearn.ensemble import RandomForestRegressor

class FeatureSelector:
    """特征选择"""
    
    @staticmethod
    def select_by_correlation(df, target_col, threshold=0.1):
        """基于相关性筛选"""
        corr = df.corr()[target_col].abs()
        selected = corr[corr > threshold].index.tolist()
        selected.remove(target_col)
        return selected
    
    @staticmethod
    def select_by_mutual_info(X, y, k=50):
        """基于互信息筛选"""
        selector = SelectKBest(score_func=mutual_info_regression, k=k)
        selector.fit(X, y)
        selected = X.columns[selector.get_support()].tolist()
        return selected
    
    @staticmethod
    def select_by_rfe(X, y, n_features=50):
        """递归特征消除"""
        estimator = RandomForestRegressor(n_estimators=50, n_jobs=-1)
        selector = RFE(estimator, n_features_to_select=n_features)
        selector.fit(X, y)
        selected = X.columns[selector.get_support()].tolist()
        return selected
    
    @staticmethod
    def select_by_importance(model, feature_names, top_n=50):
        """基于模型重要性筛选"""
        importance = pd.DataFrame({
            'feature': feature_names,
            'importance': model.feature_importances_
        }).sort_values('importance', ascending=False)
        
        return importance.head(top_n)['feature'].tolist()
```

---

## 六、模型评估

### 6.1 评估指标

```python
import numpy as np
from sklearn.metrics import (
    mean_squared_error, 
    mean_absolute_error,
    mean_absolute_percentage_error,
    r2_score,
    accuracy_score,
    precision_score,
    recall_score,
    f1_score,
    roc_auc_score
)

class ModelEvaluator:
    """模型评估器"""
    
    @staticmethod
    def regression_metrics(y_true, y_pred):
        """回归指标"""
        return {
            'RMSE': np.sqrt(mean_squared_error(y_true, y_pred)),
            'MAE': mean_absolute_error(y_true, y_pred),
            'MAPE': mean_absolute_percentage_error(y_true, y_pred),
            'R2': r2_score(y_true, y_pred),
            'RMSLE': np.sqrt(mean_squared_error(np.log1p(y_true), np.log1p(y_pred)))
        }
    
    @staticmethod
    def classification_metrics(y_true, y_pred, y_proba):
        """分类指标"""
        return {
            'Accuracy': accuracy_score(y_true, y_pred),
            'Precision': precision_score(y_true, y_pred),
            'Recall': recall_score(y_true, y_pred),
            'F1': f1_score(y_true, y_pred),
            'AUC': roc_auc_score(y_true, y_proba)
        }
    
    @staticmethod
    def top_k_accuracy(y_true, y_pred_proba, k=10):
        """Top-K 准确率（推荐场景）"""
        # 获取预测概率最高的 k 个商品
        top_k_pred = np.argsort(y_pred_proba)[-k:]
        
        # 实际爆品
        actual_hot = np.where(y_true == 1)[0]
        
        # 计算交集
        hit = len(set(top_k_pred) & set(actual_hot))
        
        return hit / k
```

### 6.2 时间序列交叉验证

```python
from sklearn.model_selection import TimeSeriesSplit

class TimeSeriesCrossValidation:
    """时间序列交叉验证"""
    
    def __init__(self, n_splits=5, gap=0):
        self.n_splits = n_splits
        self.gap = gap
        
    def split(self, X, y=None):
        """生成训练/验证索引"""
        n_samples = len(X)
        fold_size = n_samples // (self.n_splits + 1)
        
        for i in range(self.n_splits):
            train_end = (i + 1) * fold_size
            test_start = train_end + self.gap
            test_end = test_start + fold_size
            
            if test_end > n_samples:
                test_end = n_samples
            
            yield (
                np.arange(0, train_end),
                np.arange(test_start, test_end)
            )
    
    def evaluate(self, model, X, y, metrics_func):
        """评估模型"""
        scores = []
        
        for train_idx, val_idx in self.split(X):
            X_train, X_val = X.iloc[train_idx], X.iloc[val_idx]
            y_train, y_val = y.iloc[train_idx], y.iloc[val_idx]
            
            model.fit(X_train, y_train)
            y_pred = model.predict(X_val)
            
            score = metrics_func(y_val, y_pred)
            scores.append(score)
        
        return {
            'mean': np.mean(scores, axis=0),
            'std': np.std(scores, axis=0),
            'scores': scores
        }
```

---

## 七、工业实践案例

### 7.1 阿里巴巴 - 爆品推荐

**场景**：淘宝商品推荐

**技术方案**：
```
架构：
用户行为序列 ──→ DIN/DIEN ──→ 用户兴趣表示
商品特征 ──→ Embedding ──→ 商品表示
         │
         └──→ 深度网络 ──→ CTR 预测

核心算法：
1. DIN (Deep Interest Network)
   - 注意力机制捕捉用户兴趣
   
2. DIEN (Deep Interest Evolution Network)
   - GRU 建模兴趣演变

3. ESMM (Entire Space Multi-Task Model)
   - 多任务学习：CTR + CVR
```

**效果**：
- CTR 提升 10%+
- GMV 提升 5%+

### 7.2 字节跳动 - 销量预测

**场景**：抖音电商销量预测

**技术方案**：
```
特征：
- 历史销量序列
- 达人特征（粉丝、历史带货）
- 商品特征（类目、价格、图片）
- 时间特征（节假日、促销活动）

模型：
- 双塔模型（商品塔 + 达人塔）
- Transformer 编码序列
- 多模态融合（图文）

训练策略：
- 多任务学习（销量 + 转化率）
- 对比学习（相似商品）
- 增量训练
```

### 7.3 拼多多 - 需求预测

**场景**：库存备货

**技术方案**：
```
模型：LightGBM + LSTM 混合

LightGBM：处理静态特征
- 商品属性
- 历史统计

LSTM：处理时序特征
- 销量序列
- 价格序列

融合策略：
- 特征级融合：拼接后输入 MLP
- 决策级融合：加权平均预测结果
```

---

## 八、技术选型建议

### 8.1 按数据规模选择

| 数据规模 | 推荐方案 | 说明 |
|----------|----------|------|
| < 10万条 | ARIMA / Prophet | 快速验证 |
| 10万-100万 | XGBoost / LightGBM | 平衡性能和效率 |
| 100万-1000万 | LightGBM + 特征工程 | 需要分布式训练 |
| > 1000万 | 深度学习 (LSTM/Transformer) | 充分利用数据 |

### 8.2 按预测周期选择

| 预测周期 | 推荐方案 | 关键技术 |
|----------|----------|----------|
| 1-3天 | LSTM / Transformer | 捕捉短期波动 |
| 7-14天 | XGBoost + 时序特征 | 特征工程重要 |
| 30天+ | Prophet / Informer | 季节性建模 |

### 8.3 按特征丰富度选择

| 特征情况 | 推荐方案 |
|----------|----------|
| 只有销量 | Prophet / N-BEATS |
| 有外部特征 | XGBoost / LightGBM |
| 有图文特征 | 多模态模型 |
| 特征非常丰富 | Transformer |

---

## 九、未来趋势

### 9.1 大模型应用

**LLM 时序预测**：
```
思路：将时序预测转化为文本生成任务

Prompt:
"商品A过去30天的销量是：[100, 120, 115, ...]
请预测未来7天的销量："

优势：
- 无需训练，零样本预测
- 可融合文本背景信息
- 可解释性强

挑战：
- 数值精度
- 推理成本
```

**时序 Foundation Model**：
```
代表工作：
- TimeGPT (Nixtla)
- Chronos (Amazon)
- MOIRAI (Salesforce)

特点：
- 在大规模时序数据上预训练
- 零样本/少样本预测
- 可微调适配特定任务
```

### 9.2 AutoML

```
自动化机器学习流程：

1. 自动特征工程
   - 自动生成候选特征
   - 自动选择最优特征

2. 自动模型选择
   - 多模型对比
   - 超参数优化

3. AutoML 工具
   - AutoGluon
   - MLBox
   - TPOT
```

### 9.3 在线学习

```
实时更新模型：

挑战：
- 概念漂移（用户偏好变化）
- 新商品冷启动
- 数据分布变化

方案：
- 增量训练
- 在线学习算法 (FTRL, Online Random Forest)
- 模型热更新
```

---

## 十、总结

### 10.1 核心结论

1. **特征工程是关键** - 好特征比好模型更重要
2. **模型选择看场景** - 没有万能的最优模型
3. **评估要贴近业务** - 线下指标≠线上效果
4. **持续迭代优化** - 模型需要定期重训

### 10.2 推荐方案

**MVP 阶段**：
```
算法：LightGBM
特征：销量滞后 + 时间特征
评估：MAPE + Top-K 准确率
周期：2周完成
```

**进阶阶段**：
```
算法：LightGBM + LSTM 混合
特征：增加图文特征
评估：增加 A/B 测试
周期：1-2 月
```

**成熟阶段**：
```
算法：Transformer + 多模态
特征：全特征 + 外部数据
评估：在线实时评估
持续优化迭代
```

---

## 参考资料

1. [Time Series Forecasting with Transformers](https://arxiv.org/abs/2001.08317)
2. [Informer: Beyond Efficient Transformer](https://arxiv.org/abs/2012.07436)
3. [N-BEATS: Neural basis expansion analysis](https://arxiv.org/abs/1905.10437)
4. [Prophet: Automatic Forecasting Procedure](https://peerj.com/preprints/3190/)
5. [LightGBM: A Highly Efficient Gradient Boosting](https://papers.nips.cc/paper/6907-lightgbm-a-highly-efficient-gradient-boosting-decision-tree)
6. [Deep Interest Network for Click-Through Rate Prediction](https://arxiv.org/abs/1706.06978)