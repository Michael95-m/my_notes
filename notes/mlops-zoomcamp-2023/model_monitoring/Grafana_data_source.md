# Creating grafana data source

We will need to create **grafana_datasources.yaml** inside the **config** folder. A data source in Grafana represents a back-end database, such as PostgreSQL, MySQL etc.

```yaml
# config file version
apiVersion: 1

# list of datasources to insert/update
# available in the database
datasources:
  - name: PostgreSQL
    type: postgres
    access: proxy
    url: db.:5432
    database: test
    user: postgres
    secureJsonData:
      password: 'example'
    jsonData:
      sslmode: 'disable'
```

- **PostgreSQL**: A PostgreSQL database is set up as a data source. The configuration specifies that Grafana should connect to a service named **db** on port **5432**, using the database named **test**. The connection is made using the username **postgres** and the password **example**

[Prev](./Docker-compose.md) | [Next](./Baseline_monitoring_example.md)

