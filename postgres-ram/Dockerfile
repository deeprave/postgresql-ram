FROM postgres:11-alpine

COPY scripts/* /docker-entrypoint-initdb.d/

CMD ["postgres", "--max-connections=32", "--superuser-reserved-connections=2", "--shared-buffers=128kB", "--work-mem=128kB", "--temp-buffers=800kB", "--maintenance-work-mem=1024kB", "--wal-buffers=32kB", "--seq-page-cost=0.01", "--random-page-cost=0.01", "--effective-cache-size=128kB" ]
