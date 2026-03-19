# 爆品预测算法 - 详细设计文档

> 版本：v1.0  
> 日期：2026-03-19

---

## 一、概述

### 1.1 问题定义

**爆品预测**：基于历史销量、价格、评价、搜索热度、季节因素等多维度数据，预测商品在未来一段时间内是否会成为爆款。

**核心指标：**
- 预测准确率：预测为爆品且实际爆品的比例
- 召回率：实际爆品被正确预测的比例
- F1 分数：准确率和召回率的调和平均

### 1.2 技术路线

```
┌─────────────────────────────────────────────────────────────┐
│                     数据预处理                               │
│    数据清洗 │ 特征工程 │ 标签构建 │ 数据集划分               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     模型训练                                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │时序模型 │  │分类模型 │  │多模态   │  │集成模型 │        │
│  │LSTM/    │  │XGBoost/ │  │模型     │  │Ensemble │        │
│  │Transformer│ │LightGBM│  │图像+文本│  │         │        │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     预测服务                                 │
│    在线预测 │ 批量预测 │ A/B测试 │ 模型迭代                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、特征工程

### 2.1 特征维度

#### 2.1.1 销量特征

| 特征名 | 说明 | 计算 |
|--------|------|------|
| sales_1d | 当日销量 | 直接采集 |
| sales_7d | 近7天销量 | 滚动求和 |
| sales_30d | 近30天销量 | 滚动求和 |
| sales_growth_1d | 日销量增长率 | (sales_1d - sales_1d_lag1) / sales_1d_lag1 |
| sales_growth_7d | 周销量增长率 | (sales_7d - sales_7d_lag7) / sales_7d_lag7 |
| sales_acceleration | 销量加速度 | growth_1d - growth_1d_lag1 |
| sales_volatility | 销量波动率 | std(sales_7d) / mean(sales_7d) |

#### 2.1.2 价格特征

| 特征名 | 说明 |
|--------|------|
| price | 当前价格 |
| price_discount | 折扣率（原价-现价）/原价 |
| price_change_7d | 7天内价格变化 |
| price_elasticity | 价格弹性（销量变化/价格变化） |
| price_percentile | 价格分位数（同类商品中） |
| price_range_flag | 是否处于最优价格区间 |

#### 2.1.3 评价特征

| 特征名 | 说明 |
|--------|------|
| rating | 平均评分 |
| review_count | 评价数量 |
| review_growth_7d | 评价增长率 |
| positive_rate | 好评率 |
| negative_keywords_ratio | 负面关键词比例 |
| review_sentiment_score | 评价情感分数（NLP） |

#### 2.1.4 搜索热度特征

| 特征名 | 说明 |
|--------|------|
| search_volume_1d | 当日搜索量 |
| search_trend_7d | 7天搜索趋势 |
| search_rank | 搜索排名变化 |
| keyword_heat | 关键词热度 |
| related_keyword_count | 相关关键词数量 |

#### 2.1.5 时间特征

| 特征名 | 说明 |
|--------|------|
| day_of_week | 星期几 (0-6) |
| day_of_month | 月份中的日期 (1-31) |
| month | 月份 (1-12) |
| is_weekend | 是否周末 |
| is_holiday | 是否节假日 |
| season | 季节 (春/夏/秋/冬) |
| days_to_holiday | 距最近节假日天数 |

#### 2.1.6 商品特征

| 特征名 | 说明 |
|--------|------|
| category_id | 类目ID（编码） |
| category_level | 类目层级 |
| is_new_product | 是否新品 |
| days_since_launch | 上架天数 |
| has_video | 是否有视频 |
| image_count | 图片数量 |
| commission_rate | 佣金率 |

#### 2.1.7 达人特征（直播带货）

| 特征名 | 说明 |
|--------|------|
| influencer_follower_count | 达人粉丝数 |
| influencer_avg_sales | 达人平均带货量 |
| influencer_live_count | 达人直播场次 |
| influencer_category_match | 达人-类目匹配度 |
| live_viewers | 直播观看人数 |
| live_conversion_rate | 直播转化率 |

#### 2.1.8 市场特征

| 特征名 | 说明 |
|--------|------|
| category_sales_total | 类目总销量 |
| category_competition | 类目竞争度（商家数） |
| category_trend | 类目趋势 |
| hot_product_ratio | 热门商品比例 |
| market_share | 市场份额 |

### 2.2 特征处理

```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import StandardScaler, LabelEncoder
from sklearn.feature_selection import SelectKBest, mutual_info_classif

class FeatureEngineer:
    """特征工程类"""
    
    def __init__(self):
        self.scalers = {}
        self.encoders = {}
        
    def create_features(self, df: pd.DataFrame) -> pd.DataFrame:
        """创建所有特征"""
        df = df.copy()
        
        # 销量特征
        df = self._create_sales_features(df)
        
        # 价格特征
        df = self._create_price_features(df)
        
        # 评价特征
        df = self._create_review_features(df)
        
        # 时间特征
        df = self._create_time_features(df)
        
        # 滞后特征
        df = self._create_lag_features(df)
        
        # 滚动特征
        df = self._create_rolling_features(df)
        
        return df
    
    def _create_sales_features(self, df: pd.DataFrame) -> pd.DataFrame:
        """销量相关特征"""
        # 增长率
        df['sales_growth_1d'] = df.groupby('product_id')['sales_1d'].pct_change()
        df['sales_growth_7d'] = df.groupby('product_id')['sales_7d'].pct_change(7)
        
        # 加速度
        df['sales_acceleration'] = df['sales_growth_1d'] - df.groupby('product_id')['sales_growth_1d'].shift(1)
        
        # 波动率
        df['sales_volatility'] = df.groupby('product_id')['sales_1d'].transform(
            lambda x: x.rolling(7, min_periods=1).std() / x.rolling(7, min_periods=1).mean()
        )
        
        return df
    
    def _create_price_features(self, df: pd.DataFrame) -> pd.DataFrame:
        """价格相关特征"""
        # 折扣率
        df['price_discount'] = (df['original_price'] - df['price']) / df['original_price']
        
        # 价格变化
        df['price_change_7d'] = df.groupby('product_id')['price'].transform(
            lambda x: x - x.shift(7)
        )
        
        # 价格分位数
        df['price_percentile'] = df.groupby(['category_id', 'snapshot_date'])['price'].transform(
            lambda x: x.rank(pct=True)
        )
        
        return df
    
    def _create_time_features(self, df: pd.DataFrame) -> pd.DataFrame:
        """时间相关特征"""
        df['day_of_week'] = df['snapshot_date'].dt.dayofweek
        df['day_of_month'] = df['snapshot_date'].dt.day
        df['month'] = df['snapshot_date'].dt.month
        df['is_weekend'] = (df['day_of_week'] >= 5).astype(int)
        df['season'] = df['month'].apply(self._get_season)
        
        return df
    
    def _create_lag_features(self, df: pd.DataFrame, lags=[1, 3, 7, 14, 30]) -> pd.DataFrame:
        """滞后特征"""
        for lag in lags:
            df[f'sales_lag_{lag}'] = df.groupby('product_id')['sales_1d'].shift(lag)
            df[f'price_lag_{lag}'] = df.groupby('product_id')['price'].shift(lag)
            
        return df
    
    def _create_rolling_features(self, df: pd.DataFrame, windows=[7, 14, 30]) -> pd.DataFrame:
        """滚动窗口特征"""
        for window in windows:
            # 销量滚动统计
            df[f'sales_rolling_mean_{window}'] = df.groupby('product_id')['sales_1d'].transform(
                lambda x: x.rolling(window, min_periods=1).mean()
            )
            df[f'sales_rolling_std_{window}'] = df.groupby('product_id')['sales_1d'].transform(
                lambda x: x.rolling(window, min_periods=1).std()
            )
            df[f'sales_rolling_max_{window}'] = df.groupby('product_id')['sales_1d'].transform(
                lambda x: x.rolling(window, min_periods=1).max()
            )
            
        return df
    
    def normalize_features(self, df: pd.DataFrame, columns: list, fit=True) -> pd.DataFrame:
        """特征归一化"""
        for col in columns:
            if fit:
                self.scalers[col] = StandardScaler()
                df[col] = self.scalers[col].fit_transform(df[[col]])
            else:
                df[col] = self.scalers[col].transform(df[[col]])
        return df
    
    def encode_features(self, df: pd.DataFrame, columns: list, fit=True) -> pd.DataFrame:
        """类别特征编码"""
        for col in columns:
            if fit:
                self.encoders[col] = LabelEncoder()
                df[col] = self.encoders[col].fit_transform(df[col].astype(str))
            else:
                # 处理未见过的类别
                df[col] = df[col].astype(str).apply(
                    lambda x: self.encoders[col].transform([x])[0] 
                    if x in self.encoders[col].classes_ 
                    else -1
                )
        return df
    
    def select_features(self, X: pd.DataFrame, y: pd.Series, k=50) -> pd.DataFrame:
        """特征选择"""
        selector = SelectKBest(score_func=mutual_info_classif, k=k)
        X_selected = selector.fit_transform(X, y)
        
        selected_features = X.columns[selector.get_support()].tolist()
        print(f"Selected features: {selected_features}")
        
        return pd.DataFrame(X_selected, columns=selected_features, index=X.index)
    
    @staticmethod
    def _get_season(month: int) -> int:
        """获取季节"""
        if month in [3, 4, 5]:
            return 1  # 春
        elif month in [6, 7, 8]:
            return 2  # 夏
        elif month in [9, 10, 11]:
            return 3  # 秋
        else:
            return 4  # 冬
```

### 2.3 标签构建

```python
class LabelBuilder:
    """标签构建"""
    
    @staticmethod
    def build_hot_label(df: pd.DataFrame, 
                        future_days: int = 7,
                        threshold_percentile: float = 0.9) -> pd.Series:
        """
        构建爆品标签
        
        Args:
            df: 数据框
            future_days: 预测未来N天
            threshold_percentile: 销量分位数阈值
            
        Returns:
            标签序列 (1=爆品, 0=普通)
        """
        # 计算未来N天的销量
        df['future_sales'] = df.groupby('product_id')['sales_1d'].transform(
            lambda x: x.shift(-future_days).rolling(future_days).sum()
        )
        
        # 计算类目内的销量分位数
        df['sales_percentile'] = df.groupby(['category_id', 'snapshot_date'])['future_sales'].transform(
            lambda x: x.rank(pct=True)
        )
        
        # 标签：销量在类目中前10%
        label = (df['sales_percentile'] >= threshold_percentile).astype(int)
        
        return label
    
    @staticmethod
    def build_regression_label(df: pd.DataFrame, future_days: int = 7) -> pd.Series:
        """构建回归标签（预测具体销量）"""
        return df.groupby('product_id')['sales_1d'].transform(
            lambda x: x.shift(-future_days).rolling(future_days).sum()
        )
```

---

## 三、模型设计

### 3.1 时序预测模型

#### 3.1.1 LSTM 模型

```python
import torch
import torch.nn as nn
from torch.utils.data import Dataset, DataLoader

class LSTMPredictor(nn.Module):
    """LSTM 销量预测模型"""
    
    def __init__(self, 
                 input_size: int,
                 hidden_size: int = 128,
                 num_layers: int = 2,
                 dropout: float = 0.2,
                 output_size: int = 1):
        super().__init__()
        
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        
        # LSTM 层
        self.lstm = nn.LSTM(
            input_size=input_size,
            hidden_size=hidden_size,
            num_layers=num_layers,
            batch_first=True,
            dropout=dropout if num_layers > 1 else 0
        )
        
        # 全连接层
        self.fc1 = nn.Linear(hidden_size, hidden_size // 2)
        self.relu = nn.ReLU()
        self.dropout = nn.Dropout(dropout)
        self.fc2 = nn.Linear(hidden_size // 2, output_size)
        
    def forward(self, x):
        # x shape: (batch, seq_len, input_size)
        
        # LSTM
        lstm_out, (hidden, cell) = self.lstm(x)
        
        # 取最后一个时间步的输出
        last_output = lstm_out[:, -1, :]
        
        # 全连接
        out = self.fc1(last_output)
        out = self.relu(out)
        out = self.dropout(out)
        out = self.fc2(out)
        
        return out


class SalesDataset(Dataset):
    """销量数据集"""
    
    def __init__(self, data, seq_len=30):
        self.data = torch.FloatTensor(data)
        self.seq_len = seq_len
        
    def __len__(self):
        return len(self.data) - self.seq_len
    
    def __getitem__(self, idx):
        x = self.data[idx:idx+self.seq_len]
        y = self.data[idx+self.seq_len, 0]  # 假设第一列是销量
        return x, y


def train_lstm_model(model, train_loader, val_loader, epochs=100, lr=0.001):
    """训练 LSTM 模型"""
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
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
            loss = criterion(pred.squeeze(), y)
            loss.backward()
            optimizer.step()
            
            train_loss += loss.item()
        
        # 验证
        model.eval()
        val_loss = 0
        with torch.no_grad():
            for x, y in val_loader:
                x, y = x.to(device), y.to(device)
                pred = model(x)
                val_loss += criterion(pred.squeeze(), y).item()
        
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
            if patience_counter >= 10:
                print("Early stopping!")
                break
    
    # 加载最佳模型
    model.load_state_dict(torch.load('best_lstm_model.pth'))
    return model
```

#### 3.1.2 Transformer 模型

```python
class TimeSeriesTransformer(nn.Module):
    """时序 Transformer 模型"""
    
    def __init__(self,
                 input_size: int,
                 d_model: int = 64,
                 nhead: int = 4,
                 num_encoder_layers: int = 2,
                 dim_feedforward: int = 256,
                 dropout: float = 0.1,
                 output_size: int = 1):
        super().__init__()
        
        # 输入嵌入
        self.input_projection = nn.Linear(input_size, d_model)
        
        # 位置编码
        self.positional_encoding = PositionalEncoding(d_model, dropout)
        
        # Transformer 编码器
        encoder_layer = nn.TransformerEncoderLayer(
            d_model=d_model,
            nhead=nhead,
            dim_feedforward=dim_feedforward,
            dropout=dropout,
            batch_first=True
        )
        self.transformer_encoder = nn.TransformerEncoder(
            encoder_layer,
            num_layers=num_encoder_layers
        )
        
        # 输出层
        self.fc = nn.Sequential(
            nn.Linear(d_model, d_model),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(d_model, output_size)
        )
        
    def forward(self, x):
        # x shape: (batch, seq_len, input_size)
        
        # 输入投影
        x = self.input_projection(x)
        
        # 位置编码
        x = self.positional_encoding(x)
        
        # Transformer 编码
        x = self.transformer_encoder(x)
        
        # 取最后一个时间步
        x = x[:, -1, :]
        
        # 输出
        x = self.fc(x)
        
        return x


class PositionalEncoding(nn.Module):
    """位置编码"""
    
    def __init__(self, d_model: int, dropout: float = 0.1, max_len: int = 500):
        super().__init__()
        self.dropout = nn.Dropout(p=dropout)
        
        position = torch.arange(max_len).unsqueeze(1)
        div_term = torch.exp(torch.arange(0, d_model, 2) * (-math.log(10000.0) / d_model))
        
        pe = torch.zeros(1, max_len, d_model)
        pe[0, :, 0::2] = torch.sin(position * div_term)
        pe[0, :, 1::2] = torch.cos(position * div_term)
        
        self.register_buffer('pe', pe)
        
    def forward(self, x):
        # x shape: (batch, seq_len, d_model)
        x = x + self.pe[:, :x.size(1), :]
        return self.dropout(x)
```

### 3.2 分类模型

#### 3.2.1 XGBoost 分类

```python
import xgboost as xgb
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import classification_report, roc_auc_score, precision_recall_curve

class HotProductClassifier:
    """爆品分类器"""
    
    def __init__(self):
        self.model = None
        self.feature_names = None
        
    def train(self, X: pd.DataFrame, y: pd.Series, params: dict = None):
        """训练模型"""
        self.feature_names = X.columns.tolist()
        
        # 默认参数
        if params is None:
            params = {
                'objective': 'binary:logistic',
                'eval_metric': 'auc',
                'max_depth': 8,
                'learning_rate': 0.1,
                'n_estimators': 500,
                'min_child_weight': 1,
                'subsample': 0.8,
                'colsample_bytree': 0.8,
                'scale_pos_weight': len(y[y==0]) / len(y[y==1]),  # 处理不平衡
                'random_state': 42,
                'n_jobs': -1
            }
        
        # 划分数据集
        X_train, X_val, y_train, y_val = train_test_split(
            X, y, test_size=0.2, random_state=42, stratify=y
        )
        
        # 训练
        self.model = xgb.XGBClassifier(**params)
        self.model.fit(
            X_train, y_train,
            eval_set=[(X_val, y_val)],
            early_stopping_rounds=50,
            verbose=10
        )
        
        # 评估
        y_pred = self.model.predict(X_val)
        y_proba = self.model.predict_proba(X_val)[:, 1]
        
        print("\nClassification Report:")
        print(classification_report(y_val, y_pred))
        print(f"AUC: {roc_auc_score(y_val, y_proba):.4f}")
        
        return self
    
    def predict(self, X: pd.DataFrame) -> np.ndarray:
        """预测"""
        return self.model.predict(X)
    
    def predict_proba(self, X: pd.DataFrame) -> np.ndarray:
        """预测概率"""
        return self.model.predict_proba(X)[:, 1]
    
    def get_feature_importance(self, top_n: int = 20) -> pd.DataFrame:
        """获取特征重要性"""
        importance = pd.DataFrame({
            'feature': self.feature_names,
            'importance': self.model.feature_importances_
        })
        importance = importance.sort_values('importance', ascending=False)
        
        return importance.head(top_n)
    
    def plot_feature_importance(self, top_n: int = 20):
        """绘制特征重要性"""
        importance = self.get_feature_importance(top_n)
        
        plt.figure(figsize=(10, 8))
        plt.barh(importance['feature'], importance['importance'])
        plt.xlabel('Importance')
        plt.ylabel('Feature')
        plt.title('Feature Importance')
        plt.gca().invert_yaxis()
        plt.tight_layout()
        plt.show()
```

#### 3.2.2 LightGBM 分类

```python
import lightgbm as lgb

class LightGBMClassifier:
    """LightGBM 分类器"""
    
    def __init__(self):
        self.model = None
        
    def train(self, X: pd.DataFrame, y: pd.Series, params: dict = None):
        """训练"""
        if params is None:
            params = {
                'objective': 'binary',
                'metric': 'auc',
                'boosting_type': 'gbdt',
                'num_leaves': 31,
                'max_depth': -1,
                'learning_rate': 0.05,
                'n_estimators': 1000,
                'min_child_samples': 20,
                'subsample': 0.8,
                'colsample_bytree': 0.8,
                'scale_pos_weight': len(y[y==0]) / len(y[y==1]),
                'random_state': 42,
                'n_jobs': -1,
                'verbose': -1
            }
        
        X_train, X_val, y_train, y_val = train_test_split(
            X, y, test_size=0.2, random_state=42, stratify=y
        )
        
        callbacks = [
            lgb.early_stopping(stopping_rounds=50, verbose=10),
            lgb.log_evaluation(period=50)
        ]
        
        self.model = lgb.LGBMClassifier(**params)
        self.model.fit(
            X_train, y_train,
            eval_set=[(X_val, y_val)],
            callbacks=callbacks
        )
        
        # 评估
        y_pred = self.model.predict(X_val)
        y_proba = self.model.predict_proba(X_val)[:, 1]
        
        print(f"\nAUC: {roc_auc_score(y_val, y_proba):.4f}")
        
        return self
```

### 3.3 多模态模型

#### 3.3.1 图像 + 文本融合

```python
import torch
import torch.nn as nn
from transformers import BertModel, ViTModel

class MultiModalPredictor(nn.Module):
    """多模态预测模型：图像 + 文本"""
    
    def __init__(self,
                 text_model_name: str = 'bert-base-chinese',
                 image_model_name: str = 'google/vit-base-patch16-224',
                 num_numeric_features: int = 50,
                 hidden_size: int = 256,
                 output_size: int = 1):
        super().__init__()
        
        # 文本编码器
        self.text_encoder = BertModel.from_pretrained(text_model_name)
        text_hidden = self.text_encoder.config.hidden_size  # 768
        
        # 图像编码器
        self.image_encoder = ViTModel.from_pretrained(image_model_name)
        image_hidden = self.image_encoder.config.hidden_size  # 768
        
        # 数值特征处理
        self.numeric_encoder = nn.Sequential(
            nn.Linear(num_numeric_features, hidden_size),
            nn.ReLU(),
            nn.Dropout(0.2)
        )
        
        # 融合层
        fusion_input_size = text_hidden + image_hidden + hidden_size
        self.fusion = nn.Sequential(
            nn.Linear(fusion_input_size, hidden_size * 2),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(hidden_size * 2, hidden_size),
            nn.ReLU(),
            nn.Dropout(0.2)
        )
        
        # 输出层
        self.output = nn.Linear(hidden_size, output_size)
        
    def forward(self, text_input, image_input, numeric_input):
        # 文本编码
        text_output = self.text_encoder(**text_input)
        text_embedding = text_output.last_hidden_state[:, 0, :]  # [CLS] token
        
        # 图像编码
        image_output = self.image_encoder(**image_input)
        image_embedding = image_output.last_hidden_state[:, 0, :]  # [CLS] token
        
        # 数值特征编码
        numeric_embedding = self.numeric_encoder(numeric_input)
        
        # 拼接融合
        fused = torch.cat([text_embedding, image_embedding, numeric_embedding], dim=-1)
        
        # 融合层
        fused = self.fusion(fused)
        
        # 输出
        output = self.output(fused)
        
        return output


class ProductMultiModalDataset(Dataset):
    """商品多模态数据集"""
    
    def __init__(self, df, tokenizer, image_processor, numeric_features):
        self.df = df
        self.tokenizer = tokenizer
        self.image_processor = image_processor
        self.numeric_features = numeric_features
        
    def __len__(self):
        return len(self.df)
    
    def __getitem__(self, idx):
        row = self.df.iloc[idx]
        
        # 文本
        text = row['title'] + ' ' + row['description']
        text_input = self.tokenizer(
            text,
            padding='max_length',
            truncation=True,
            max_length=128,
            return_tensors='pt'
        )
        
        # 图像
        image = Image.open(row['image_path']).convert('RGB')
        image_input = self.image_processor(images=image, return_tensors='pt')
        
        # 数值特征
        numeric = torch.FloatTensor(row[self.numeric_features].values)
        
        # 标签
        label = torch.FloatTensor([row['label']])
        
        return {
            'text_input': {k: v.squeeze(0) for k, v in text_input.items()},
            'image_input': {k: v.squeeze(0) for k, v in image_input.items()},
            'numeric_input': numeric,
            'label': label
        }
```

### 3.4 集成模型

```python
from sklearn.ensemble import VotingClassifier, StackingClassifier
from sklearn.linear_model import LogisticRegression

class EnsemblePredictor:
    """集成预测模型"""
    
    def __init__(self):
        self.models = {}
        self.ensemble = None
        
    def add_model(self, name: str, model):
        """添加模型"""
        self.models[name] = model
        
    def build_voting_ensemble(self, voting: str = 'soft'):
        """构建投票集成"""
        estimators = [(name, model) for name, model in self.models.items()]
        self.ensemble = VotingClassifier(estimators=estimators, voting=voting)
        return self.ensemble
    
    def build_stacking_ensemble(self):
        """构建堆叠集成"""
        estimators = [(name, model) for name, model in self.models.items()]
        self.ensemble = StackingClassifier(
            estimators=estimators,
            final_estimator=LogisticRegression(),
            cv=5
        )
        return self.ensemble
    
    def train(self, X, y):
        """训练集成模型"""
        if self.ensemble is None:
            raise ValueError("Please build ensemble first!")
        
        self.ensemble.fit(X, y)
        return self
    
    def predict(self, X):
        """预测"""
        return self.ensemble.predict(X)
    
    def predict_proba(self, X):
        """预测概率"""
        return self.ensemble.predict_proba(X)[:, 1]
```

---

## 四、模型训练 Pipeline

### 4.1 完整训练流程

```python
class ModelTrainingPipeline:
    """模型训练 Pipeline"""
    
    def __init__(self, config: dict):
        self.config = config
        self.feature_engineer = FeatureEngineer()
        self.label_builder = LabelBuilder()
        self.model = None
        self.best_threshold = 0.5
        
    def prepare_data(self, df: pd.DataFrame) -> tuple:
        """准备数据"""
        # 特征工程
        df = self.feature_engineer.create_features(df)
        
        # 标签构建
        y = self.label_builder.build_hot_label(
            df, 
            future_days=self.config['future_days'],
            threshold_percentile=self.config['threshold_percentile']
        )
        
        # 特征选择
        feature_cols = [col for col in df.columns 
                       if col not in ['product_id', 'snapshot_date', 'label', 'future_sales', 'sales_percentile']]
        X = df[feature_cols].copy()
        
        # 处理缺失值
        X = X.fillna(0)
        
        # 处理无穷值
        X = X.replace([np.inf, -np.inf], 0)
        
        # 归一化
        numeric_cols = X.select_dtypes(include=[np.number]).columns.tolist()
        X = self.feature_engineer.normalize_features(X, numeric_cols, fit=True)
        
        # 删除标签对应的行（未来数据）
        valid_idx = ~y.isna()
        X = X[valid_idx]
        y = y[valid_idx]
        
        return X, y
    
    def train(self, X: pd.DataFrame, y: pd.Series) -> dict:
        """训练模型"""
        from sklearn.model_selection import cross_val_score, StratifiedKFold
        
        # 划分数据集
        X_train, X_test, y_train, y_test = train_test_split(
            X, y, test_size=0.2, random_state=42, stratify=y
        )
        
        # 训练多个模型
        results = {}
        
        # XGBoost
        print("Training XGBoost...")
        xgb_clf = HotProductClassifier()
        xgb_clf.train(X_train, y_train)
        results['xgboost'] = self._evaluate(xgb_clf, X_test, y_test)
        
        # LightGBM
        print("Training LightGBM...")
        lgb_clf = LightGBMClassifier()
        lgb_clf.train(X_train, y_train)
        results['lightgbm'] = self._evaluate(lgb_clf, X_test, y_test)
        
        # 集成模型
        print("Training Ensemble...")
        ensemble = EnsemblePredictor()
        ensemble.add_model('xgboost', xgb_clf.model)
        ensemble.add_model('lightgbm', lgb_clf.model)
        ensemble.build_stacking_ensemble()
        ensemble.train(X_train, y_train)
        results['ensemble'] = self._evaluate(ensemble, X_test, y_test)
        
        # 选择最佳模型
        best_model_name = max(results, key=lambda x: results[x]['auc'])
        print(f"\nBest model: {best_model_name}")
        
        self.model = {
            'xgboost': xgb_clf,
            'lightgbm': lgb_clf,
            'ensemble': ensemble
        }[best_model_name]
        
        # 优化阈值
        self.best_threshold = self._optimize_threshold(X_test, y_test)
        
        return results
    
    def _evaluate(self, model, X, y) -> dict:
        """评估模型"""
        y_pred = model.predict(X)
        y_proba = model.predict_proba(X)
        
        return {
            'auc': roc_auc_score(y, y_proba),
            'precision': precision_score(y, y_pred),
            'recall': recall_score(y, y_pred),
            'f1': f1_score(y, y_pred)
        }
    
    def _optimize_threshold(self, X, y) -> float:
        """优化分类阈值"""
        y_proba = self.model.predict_proba(X)
        
        precisions, recalls, thresholds = precision_recall_curve(y, y_proba)
        f1_scores = 2 * (precisions * recalls) / (precisions + recalls + 1e-10)
        
        best_idx = np.argmax(f1_scores)
        best_threshold = thresholds[best_idx]
        
        print(f"Best threshold: {best_threshold:.4f}, F1: {f1_scores[best_idx]:.4f}")
        
        return best_threshold
    
    def predict(self, X: pd.DataFrame) -> pd.DataFrame:
        """预测"""
        # 特征处理
        numeric_cols = X.select_dtypes(include=[np.number]).columns.tolist()
        X = self.feature_engineer.normalize_features(X, numeric_cols, fit=False)
        
        # 预测概率
        proba = self.model.predict_proba(X)
        
        # 预测标签
        pred = (proba >= self.best_threshold).astype(int)
        
        # 结果
        result = pd.DataFrame({
            'hot_probability': proba,
            'is_hot': pred
        })
        
        return result
    
    def save(self, path: str):
        """保存模型"""
        import joblib
        
        joblib.dump({
            'model': self.model,
            'feature_engineer': self.feature_engineer,
            'best_threshold': self.best_threshold,
            'config': self.config
        }, path)
        
    @classmethod
    def load(cls, path: str):
        """加载模型"""
        import joblib
        
        data = joblib.load(path)
        
        pipeline = cls(data['config'])
        pipeline.model = data['model']
        pipeline.feature_engineer = data['feature_engineer']
        pipeline.best_threshold = data['best_threshold']
        
        return pipeline
```

---

## 五、在线预测服务

### 5.1 服务架构

```java
@Service
@Slf4j
public class HotProductPredictionService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ProductSalesSnapshotRepository snapshotRepository;
    
    @Autowired
    private HotProductPredictionRepository predictionRepository;
    
    @Value("${model.path}")
    private String modelPath;
    
    private ModelTrainingPipeline pipeline;
    
    @PostConstruct
    public void init() {
        // 加载模型
        pipeline = ModelTrainingPipeline.load(modelPath);
    }
    
    /**
     * 预测单个商品
     */
    public HotProductPrediction predict(Long productId) {
        // 获取商品特征
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new NotFoundException("商品不存在"));
        
        // 获取历史销量快照
        List<ProductSalesSnapshot> snapshots = snapshotRepository
            .findByProductIdOrderBySnapshotDateDesc(productId, PageRequest.of(0, 30));
        
        if (snapshots.size() < 7) {
            throw new BusinessException("数据不足，无法预测");
        }
        
        // 构建特征
        Map<String, Object> features = buildFeatures(product, snapshots);
        
        // 转换为 DataFrame
        DataFrame df = convertToDataFrame(features);
        
        // 预测
        DataFrame result = pipeline.predict(df);
        
        // 构建结果
        HotProductPrediction prediction = new HotProductPrediction();
        prediction.setProductId(productId);
        prediction.setPredictDate(LocalDate.now().plusDays(7));
        prediction.setPredictScore(result.getDouble("hot_probability"));
        prediction.setConfidence(calculateConfidence(features));
        prediction.setFactors(extractFactors(features));
        prediction.setModelVersion(pipeline.getConfig().get("model_version"));
        
        // 保存
        return predictionRepository.save(prediction);
    }
    
    /**
     * 批量预测
     */
    @Async
    public void batchPredict(List<Long> productIds) {
        for (Long productId : productIds) {
            try {
                predict(productId);
            } catch (Exception e) {
                log.error("预测失败: productId={}", productId, e);
            }
        }
    }
    
    /**
     * 每日批量预测任务
     */
    @Scheduled(cron = "0 0 6 * * ?") // 每天早上6点
    public void dailyPrediction() {
        // 获取活跃商品
        List<Long> activeProductIds = productRepository.findActiveProductIds();
        
        log.info("开始每日预测，商品数: {}", activeProductIds.size());
        
        batchPredict(activeProductIds);
        
        log.info("每日预测完成");
    }
    
    /**
     * 获取爆品推荐列表
     */
    public List<HotProductPrediction> getTopHotPredictions(
            String categoryId, 
            LocalDate predictDate, 
            int limit) {
        
        Specification<HotProductPrediction> spec = Specification
            .where(HotProductPredictionSpec.predictDateEq(predictDate))
            .and(HotProductPredictionSpec.categoryEq(categoryId));
        
        return predictionRepository.findAll(spec, 
            PageRequest.of(0, limit, Sort.by("predictScore").descending())
        ).getContent();
    }
}
```

### 5.2 API 接口

```java
@RestController
@RequestMapping("/api/prediction")
@Api(tags = "爆品预测")
public class PredictionController {
    
    @Autowired
    private HotProductPredictionService predictionService;
    
    @PostMapping("/predict/{productId}")
    @ApiOperation("预测单个商品")
    public Result<HotProductPrediction> predict(@PathVariable Long productId) {
        HotProductPrediction prediction = predictionService.predict(productId);
        return Result.success(prediction);
    }
    
    @PostMapping("/batch")
    @ApiOperation("批量预测")
    public Result<Void> batchPredict(@RequestBody List<Long> productIds) {
        predictionService.batchPredict(productIds);
        return Result.success();
    }
    
    @GetMapping("/hot-products")
    @ApiOperation("获取爆品推荐列表")
    public Result<List<HotProductPrediction>> getHotProducts(
            @RequestParam(required = false) String categoryId,
            @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate date,
            @RequestParam(defaultValue = "20") int limit) {
        
        LocalDate predictDate = date != null ? date : LocalDate.now().plusDays(7);
        List<HotProductPrediction> predictions = predictionService
            .getTopHotPredictions(categoryId, predictDate, limit);
        
        return Result.success(predictions);
    }
    
    @GetMapping("/product/{productId}/history")
    @ApiOperation("获取商品预测历史")
    public Result<List<HotProductPrediction>> getPredictionHistory(
            @PathVariable Long productId,
            @RequestParam(defaultValue = "30") int days) {
        
        LocalDate startDate = LocalDate.now().minusDays(days);
        List<HotProductPrediction> history = predictionService
            .getPredictionHistory(productId, startDate);
        
        return Result.success(history);
    }
}
```

---

## 六、模型评估与迭代

### 6.1 评估指标

```python
class ModelEvaluator:
    """模型评估器"""
    
    @staticmethod
    def evaluate(y_true, y_pred, y_proba):
        """综合评估"""
        from sklearn.metrics import (
            accuracy_score, precision_score, recall_score, f1_score,
            roc_auc_score, average_precision_score, confusion_matrix,
            classification_report
        )
        
        metrics = {
            'accuracy': accuracy_score(y_true, y_pred),
            'precision': precision_score(y_true, y_pred),
            'recall': recall_score(y_true, y_pred),
            'f1': f1_score(y_true, y_pred),
            'auc': roc_auc_score(y_true, y_proba),
            'auprc': average_precision_score(y_true, y_proba)
        }
        
        print("Classification Report:")
        print(classification_report(y_true, y_pred))
        
        print("\nConfusion Matrix:")
        print(confusion_matrix(y_true, y_pred))
        
        return metrics
    
    @staticmethod
    def plot_roc_curve(y_true, y_proba):
        """绘制 ROC 曲线"""
        from sklearn.metrics import roc_curve, auc
        
        fpr, tpr, _ = roc_curve(y_true, y_proba)
        roc_auc = auc(fpr, tpr)
        
        plt.figure()
        plt.plot(fpr, tpr, color='darkorange', lw=2, 
                label=f'ROC curve (AUC = {roc_auc:.2f})')
        plt.plot([0, 1], [0, 1], color='navy', lw=2, linestyle='--')
        plt.xlim([0.0, 1.0])
        plt.ylim([0.0, 1.05])
        plt.xlabel('False Positive Rate')
        plt.ylabel('True Positive Rate')
        plt.title('Receiver Operating Characteristic')
        plt.legend(loc="lower right")
        plt.show()
    
    @staticmethod
    def plot_precision_recall_curve(y_true, y_proba):
        """绘制 PR 曲线"""
        from sklearn.metrics import precision_recall_curve, average_precision_score
        
        precision, recall, _ = precision_recall_curve(y_true, y_proba)
        auprc = average_precision_score(y_true, y_proba)
        
        plt.figure()
        plt.plot(recall, precision, color='blue', lw=2,
                label=f'PR curve (AUPRC = {auprc:.2f})')
        plt.xlim([0.0, 1.0])
        plt.ylim([0.0, 1.05])
        plt.xlabel('Recall')
        plt.ylabel('Precision')
        plt.title('Precision-Recall Curve')
        plt.legend(loc="lower left")
        plt.show()
    
    @staticmethod
    def calibration_analysis(y_true, y_proba):
        """校准分析"""
        from sklearn.calibration import calibration_curve
        
        prob_true, prob_pred = calibration_curve(y_true, y_proba, n_bins=10)
        
        plt.figure()
        plt.plot(prob_pred, prob_true, marker='o')
        plt.plot([0, 1], [0, 1], linestyle='--')
        plt.xlabel('Mean predicted probability')
        plt.ylabel('Fraction of positives')
        plt.title('Calibration Curve')
        plt.show()
```

### 6.2 模型监控

```java
@Service
public class ModelMonitor {
    
    /**
     * 监控预测准确率
     */
    @Scheduled(cron = "0 0 0 * * ?") // 每天凌晨
    public void monitorAccuracy() {
        // 获取7天前的预测
        LocalDate predictDate = LocalDate.now().minusDays(7);
        List<HotProductPrediction> predictions = predictionRepository
            .findByPredictDate(predictDate);
        
        int correct = 0;
        int total = 0;
        
        for (HotProductPrediction pred : predictions) {
            // 获取实际销量
            Product product = productRepository.findById(pred.getProductId()).orElse(null);
            if (product == null) continue;
            
            // 计算实际是否爆品
            double actualSalesPercentile = calculateSalesPercentile(
                product.getId(), 
                product.getCategoryId(), 
                predictDate
            );
            boolean actualHot = actualSalesPercentile >= 0.9;
            
            // 判断预测是否正确
            if (pred.getIsHot() == actualHot) {
                correct++;
            }
            total++;
        }
        
        double accuracy = total > 0 ? (double) correct / total : 0;
        
        log.info("预测准确率: {:.2%} ({}/{})", accuracy, correct, total);
        
        // 发送告警
        if (accuracy < 0.6) {
            alertService.sendAlert("预测准确率低于60%，需要重新训练模型！");
        }
        
        // 保存监控数据
        ModelMetrics metrics = new ModelMetrics();
        metrics.setDate(LocalDate.now());
        metrics.setAccuracy(accuracy);
        metrics.setSampleCount(total);
        metricsRepository.save(metrics);
    }
}
```

---

## 七、总结

### 7.1 技术选型总结

| 任务 | 推荐模型 | 说明 |
|------|----------|------|
| 销量时序预测 | LSTM / Transformer | 捕捉时间依赖 |
| 爆品分类 | XGBoost / LightGBM | 处理结构化数据，可解释性强 |
| 多模态融合 | BERT + ViT | 图文结合预测 |
| 集成学习 | Stacking | 提升准确率 |

### 7.2 关键成功因素

1. **特征工程** - 好特征比好模型更重要
2. **标签定义** - 爆品的标准要明确
3. **数据质量** - 数据量大、覆盖全面
4. **持续迭代** - 定期重训练、监控效果

### 7.3 后续优化方向

1. 引入更多外部特征（热搜、舆情）
2. 尝试更复杂的模型架构
3. 优化预测延迟（实时特征）
4. A/B 测试不同模型版本