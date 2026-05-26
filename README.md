# thmi-atrium-ljungberg

Installation

1. Go to folder `.docker`
2. Run MariaDB, Clickhouse and RabbitMQ containers: `docker-compose -f env.yml up -d`
3. Populate database `docker run --network=dev -it --rm engine-utils`
4. 