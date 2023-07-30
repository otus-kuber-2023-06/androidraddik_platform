FROM python:3.7
COPY templates ./templates
COPY mysql-operator.py ./mysql-operator.py
RUN pip install kopf==1.34.0 kubernetes==20.13.0 pyyaml jinja2==3.0.1
CMD kopf run /mysql-operator.py