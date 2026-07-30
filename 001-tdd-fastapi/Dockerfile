FROM python:3.14-slim-bookworm

# set working directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install system dependencies
RUN apt-get update \
   && apt-get -y install netcat-traditional gcc postgresql \
   && apt-get clean

# install python dependencies
RUN pip install --upgrade pip
COPY pyproject.toml ./
RUN pip install --upgrade "uvicorn[standard]>=0.34.1"
RUN pip install .[dev]

# add app
COPY . .

# add entrypoint.sh
COPY ./entrypoint.sh .
RUN chmod +x /usr/src/app/entrypoint.sh

# run entrypoint.sh
ENTRYPOINT ["/usr/src/app/entrypoint.sh"]
