# ML Engineering Patterns

Production-ready patterns for machine learning systems.

## Data Pipeline Patterns

### Train/Test Split - Prevent Data Leakage

```python
# BAD: Fitting on all data causes leakage
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)  # Leaks test info into training
X_train, X_test = train_test_split(X_scaled)

# GOOD: Fit only on training data
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)  # Transform only, no fit
```

### Time Series Split - Respect Temporal Order

```python
# BAD: Random split breaks temporal dependencies
X_train, X_test = train_test_split(time_series_data)

# GOOD: Use time-based split
from sklearn.model_selection import TimeSeriesSplit

tscv = TimeSeriesSplit(n_splits=5)
for train_idx, test_idx in tscv.split(X):
    X_train, X_test = X[train_idx], X[test_idx]
    # Train always comes before test temporally
```

### Feature Engineering Pipeline

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer

# Define preprocessing for different column types
numeric_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler())
])

categorical_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='constant', fill_value='missing')),
    ('encoder', OneHotEncoder(handle_unknown='ignore'))
])

preprocessor = ColumnTransformer([
    ('num', numeric_transformer, numeric_features),
    ('cat', categorical_transformer, categorical_features)
])

# Full pipeline prevents leakage
model_pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('classifier', RandomForestClassifier())
])

# Fit on training data only
model_pipeline.fit(X_train, y_train)
```

## Model Training Patterns

### Reproducibility

```python
import random
import numpy as np
import torch

def set_seed(seed: int = 42):
    """Set all random seeds for reproducibility."""
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

# Call at start of training
set_seed(42)
```

### Early Stopping

```python
class EarlyStopping:
    def __init__(self, patience: int = 10, min_delta: float = 0.0):
        self.patience = patience
        self.min_delta = min_delta
        self.counter = 0
        self.best_loss = float('inf')
        self.should_stop = False
    
    def __call__(self, val_loss: float) -> bool:
        if val_loss < self.best_loss - self.min_delta:
            self.best_loss = val_loss
            self.counter = 0
        else:
            self.counter += 1
            if self.counter >= self.patience:
                self.should_stop = True
        return self.should_stop

# Usage
early_stopping = EarlyStopping(patience=10)
for epoch in range(max_epochs):
    train_loss = train_epoch(model, train_loader)
    val_loss = validate(model, val_loader)
    
    if early_stopping(val_loss):
        print(f"Early stopping at epoch {epoch}")
        break
```

### Gradient Clipping

```python
# Prevent exploding gradients
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

# Or clip by value
torch.nn.utils.clip_grad_value_(model.parameters(), clip_value=0.5)
```

## Numerical Stability

### Log-Sum-Exp Trick

```python
# BAD: Overflow risk
def softmax_naive(x):
    exp_x = np.exp(x)
    return exp_x / np.sum(exp_x)

# GOOD: Numerically stable
def softmax_stable(x):
    exp_x = np.exp(x - np.max(x))
    return exp_x / np.sum(exp_x)
```

### Safe Division

```python
# BAD: Division by zero
result = numerator / denominator

# GOOD: Add epsilon
eps = 1e-8
result = numerator / (denominator + eps)
```

### Mixed Precision Training

```python
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()

for batch in dataloader:
    optimizer.zero_grad()
    
    with autocast():
        outputs = model(inputs)
        loss = criterion(outputs, targets)
    
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

## Model Evaluation

### Cross-Validation

```python
from sklearn.model_selection import cross_val_score, StratifiedKFold

# Stratified for classification (maintains class distribution)
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X, y, cv=cv, scoring='f1_weighted')

print(f"CV Score: {scores.mean():.3f} (+/- {scores.std() * 2:.3f})")
```

### Confidence Intervals

```python
from scipy import stats

def bootstrap_ci(data, n_bootstrap=1000, ci=0.95):
    """Calculate bootstrap confidence interval."""
    bootstrapped = []
    for _ in range(n_bootstrap):
        sample = np.random.choice(data, size=len(data), replace=True)
        bootstrapped.append(np.mean(sample))
    
    lower = np.percentile(bootstrapped, (1 - ci) / 2 * 100)
    upper = np.percentile(bootstrapped, (1 + ci) / 2 * 100)
    return lower, upper
```

## Model Serialization

### Safe Model Loading

```python
import torch
import pickle

# BAD: pickle is insecure
model = pickle.load(open('model.pkl', 'rb'))

# GOOD: Use torch.load with weights_only for PyTorch
model = MyModel()
model.load_state_dict(torch.load('model.pt', weights_only=True))

# GOOD: Use joblib for sklearn
from joblib import dump, load
dump(model, 'model.joblib')
model = load('model.joblib')
```

### Model Versioning

```python
import hashlib
from datetime import datetime

def save_model_with_metadata(model, path, metrics):
    """Save model with versioning metadata."""
    metadata = {
        'timestamp': datetime.utcnow().isoformat(),
        'metrics': metrics,
        'model_hash': hashlib.md5(
            pickle.dumps(model.state_dict())
        ).hexdigest()
    }
    
    torch.save({
        'model_state_dict': model.state_dict(),
        'metadata': metadata
    }, path)
```

## Inference Patterns

### Batch Prediction

```python
def predict_batch(model, data, batch_size=32):
    """Memory-efficient batch prediction."""
    model.eval()
    predictions = []
    
    with torch.no_grad():
        for i in range(0, len(data), batch_size):
            batch = data[i:i + batch_size]
            pred = model(batch)
            predictions.append(pred.cpu())
    
    return torch.cat(predictions)
```

### Model Caching

```python
from functools import lru_cache

@lru_cache(maxsize=1)
def get_model():
    """Load model once and cache."""
    model = load_model('model.pt')
    model.eval()
    return model

def predict(input_data):
    model = get_model()
    return model(input_data)
```

## Common Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Fitting scaler on all data | Data leakage | Fit on train only |
| Random split for time series | Future leakage | Time-based split |
| No seed setting | Non-reproducible | Set all seeds |
| Naive softmax | Numerical overflow | Log-sum-exp trick |
| pickle for models | Security risk | Use safe loaders |
| Loading model per request | Slow inference | Cache model |
| No gradient clipping | Exploding gradients | Clip gradients |
