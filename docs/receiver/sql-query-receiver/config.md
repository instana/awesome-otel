1. Under receivers: (Added the sqlquery Receiver)

  # SQL Query receiver
  # Runs custom SQL against a database and emits the results as metrics
  sqlquery:
    driver: sqlserver
    host: 127.0.0.1
    port: 1433
    database: testdb
    username: instana_otel
    password: InstanaPassword123!
    collection_interval: 10s
    queries:
      # Total completed orders and revenue
      - sql: "SELECT COUNT(*) AS total_orders, SUM(amount) AS total_revenue FROM orders WHERE status = 'COMPLETED';"
        metrics:
          - metric_name: sqlserver.custom.orders.completed_count
            value_column: total_orders
            value_type: int
            data_type: gauge
          - metric_name: sqlserver.custom.orders.completed_revenue
            value_column: total_revenue
            value_type: double
            data_type: gauge
      # Active user connections
      - sql: "SELECT COUNT(*) AS connection_count FROM sys.dm_exec_sessions WHERE is_user_process = 1"
        metrics:
          - metric_name: sqlserver.user.connection.count
            value_column: connection_count
            value_type: int
            data_type: gauge


2. Under service.pipelines: (Added the metrics/sqlquery Pipeline)
    # Metrics data pipeline for SQL queries
    metrics/sqlquery:
      receivers: [sqlquery]
      processors: [resource/host, batch]
      exporters: [otlphttp/exporter]