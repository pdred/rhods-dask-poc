FROM daskdev/dask:latest
RUN pip install dask distributed --upgrade
CMD ["dask-worker", "tcp://dask-scheduler:8786"]
