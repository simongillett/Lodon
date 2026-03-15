# Lodon Trading System Architecture

```mermaid
flowchart TB
    subgraph DataSources["Data Sources"]
        FMP["FMP API\n(Financial Modeling Prep)"]
    end

    subgraph DataPipeline["Data Pipeline"]
        direction TB
        PricesDB[(company_prices_daily.db\nDuckDB)]
        MetricsDB[(company_metrics_1yr.db\nDuckDB)]
        DailyUpdate["daily_update.py\n6am daily"]
        FetchMetrics["fetch_key_metrics_1yr.py\nSunday 8pm"]
    end

    subgraph FactorModel["Factor Model"]
        direction TB
        Backtest["normalization_weighting_Zipline.py\nSunday 8pm"]
        FeatureSelect["Feature Selection\nVariance threshold\nCorrelation filter"]
        Scaler["StandardScaler\nNormalization"]
        FactorArtifacts["DynamicWeeklyFactors\nfactors_latest.json\nscaler_latest.joblib"]
    end

    subgraph SignalGen["Signal Generation"]
        direction TB
        DailySignalGen["daily_signal_generator.py\n8am Mon-Fri"]
        SignalFiles["daily_signals\nsignals_YYYYMMDD.json"]
        SignalPub["signal_publisher.py"]
    end

    subgraph MessageQueue["AWS SQS FIFO"]
        PaperQ["trading-paper-signals.fifo"]
        LiveQ["trading-live-signals.fifo"]
    end

    subgraph Execution["Execution (EC2)"]
        direction TB
        ExecEngine["execution_engine.py\n9:25am Mon-Fri"]
        IBGateway["IB Gateway\nlocalhost:4001/4002"]
        IBC["IBC Controller\nlogin 2FA restarts"]
        EnvConfig["/opt/trading/.env"]
        TradeLogs["logs\nengine_YYYYMMDD.log\ntrades.json"]
    end

    subgraph Broker["IBKR"]
        IBKRServers["IBKR Servers"]
    end

    subgraph Positions["Position Management"]
        Portfolio["Portfolio\nMax 15 positions\nScore-weighted sizing\n2x leverage"]
    end

    FMP -->|prices| DailyUpdate
    FMP -->|metrics| FetchMetrics
    DailyUpdate --> PricesDB
    FetchMetrics --> MetricsDB

    PricesDB --> Backtest
    MetricsDB --> Backtest
    Backtest --> FeatureSelect
    FeatureSelect -->|21 factors| Scaler
    Scaler --> FactorArtifacts

    FactorArtifacts --> DailySignalGen
    PricesDB --> DailySignalGen
    MetricsDB --> DailySignalGen
    DailySignalGen --> SignalFiles
    DailySignalGen --> SignalPub
    SignalPub -->|paper| PaperQ
    SignalPub -->|live| LiveQ

    PaperQ --> ExecEngine
    LiveQ --> ExecEngine
    EnvConfig --> ExecEngine
    ExecEngine -->|TWS API| IBGateway
    IBC -->|manages| IBGateway
    IBGateway --> IBKRServers
    ExecEngine --> TradeLogs

    IBKRServers --> Portfolio

    classDef source fill:#e1f5fe,stroke:#01579b
    classDef storage fill:#fff3e0,stroke:#e65100
    classDef model fill:#f3e5f5,stroke:#7b1fa2
    classDef signal fill:#e8f5e9,stroke:#2e7d32
    classDef queue fill:#fce4ec,stroke:#c2185b
    classDef exec fill:#fff8e1,stroke:#f57f17
    classDef broker fill:#e0f2f1,stroke:#00695c

    class FMP source
    class PricesDB,MetricsDB storage
    class Backtest,FeatureSelect,Scaler,FactorArtifacts model
    class DailySignalGen,SignalFiles,SignalPub signal
    class PaperQ,LiveQ queue
    class ExecEngine,IBGateway,IBC,EnvConfig,TradeLogs exec
    class IBKRServers,Portfolio broker
