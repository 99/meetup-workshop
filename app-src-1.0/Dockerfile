FROM python:3.6-alpine

ADD . /src/hello/
WORKDIR /src/hello
RUN pip install -r requirements.txt

EXPOSE 5000
CMD python hello.py