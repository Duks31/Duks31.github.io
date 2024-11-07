---
title: "Car Price Prediction"
excerpt: "Building machine learning systems"
collection: portfolio
date: 2024-08-01
---

## ML: From Data scraping to Deployment

## Building a Car Prediction Model

Machine learning systems are not built from Jupyter notebooks, for the most part of it they are much complex systems.

In my self-learning journey, I wanted to learn about putting different systems and technologies together. So, I used a simple, not to complicated problem — car price prediction- to understand how it done in a small scale. I would be explaining the whole process below

So, we would be going through the workflow below.

Project code: [https://github.com/Duks31/car_price-prediction](https://github.com/Duks31/car_price-prediction)

![Project workflow (image by author)](https://cdn-images-1.medium.com/max/7622/1*jVYjxaXZtEQFJfJJROVJcg.png)

## **Data Scarping**

![Data Scraping workflow (image by author)](https://cdn-images-1.medium.com/max/2000/1*MvNxg9J1ygsVPrDYcFcYFg.png)

First, I scraped the data from a car website ([car45](https://www.cars45.com/)) using scrapy. In scrapy there have what they call spiders, which is used to extract data from websites by specifying the URLs to crawl, the data to collect, and how to process and store the extracted information. You can definitely have more than one scrapy spider.

The whole scrapy code can be found here: [Duks31/car-scraper: car scraper repo (github.com)](https://github.com/Duks31/car-scraper)

Let's start with the spider.

```python
    import scrapy
    from car_scraper.items import CarScraperItem
    
    class CarscraperSpider(scrapy.Spider):
        name = "carScraper"
        allowed_domains = ["www.cars45.com"]
        start_urls = ["https://www.cars45.com/listing"]
    
        def parse(self, response):
            cars = response.css("section.cars-grid.grid a.car-feature--wide-mobile")
            for car in cars:
                car_url = car.css("::attr(href)").get()
                yield response.follow(car_url, callback=self.parse_car_page)
    
            # next page
            next_page = response.css('a.pagination__next::attr(href)').get()
            if next_page:
                yield response.follow(next_page, callback = self.parse)
    
        def parse_car_page(self, response): 
            general_info = response.xpath('//div[@class="general-info grid"]/div')
            tags = response.css("div.main-details__tags span::text").getall()
            svg_info = response.css("div.svg.flex > div")
        
            car_item = CarScraperItem()
            car_item["car_id"] = response.url.split("/")[-1]
            car_item["city"] = response.css("p.main-details__region::text").get()
            car_item["price"] = response.css("h5.main-details__name__price::text").get()
            car_item["condition"] = tags[0] if len(tags) > 0 else None
            car_item["transmission"] = tags[1] if len(tags) > 1 else None
            car_item["mileage"] = tags[2] if len(tags) > 2 else None
            for svg in svg_info:
                alt_text = svg.css("img::attr(alt)").get()
                value = svg.css("span.tab-content__svg__title::text").get()
                if alt_text == "Body":
                    car_item["body_type"] = value
                elif alt_text == "Fuel":
                    car_item["fuel_type"] = value
                elif alt_text == "Transmission":
                    car_item["transmission"] = value
    
            for info in general_info:
                name = info.xpath('.//span[@class="general-info__value"]/text()').get()
                value = info.xpath('.//p[@class="general-info__name"]/text()').get()
                if name and value:
                    name = name.strip()
                    value = value.strip()
                    if name == "Make":
                        car_item["make"] = value
                    elif name == "Model":
                        car_item["model"] = value
                    elif name == "Year of manufacture":
                        car_item["year_of_manufacture"] = value
                    elif name == "Colour":
                        car_item["color"] = value
                    elif name == "Engine Size":
                        car_item["engine_size"] = value
                    elif name == "Registered city":
                        car_item["registered_city"] = value
                    elif name == "Selling Condition":
                        car_item["selling_condition"] = value
                    elif name == "Bought Condition":
                        car_item["bought_condition"] = value
    
            yield car_item
```

The spider can be found in carScraper.py The script goes to the listing page and then each car’s page and for each car it gets the make, model, year of manufacture, colour, engine size, registered city, selling condition, bought condition, body type, fuel type, transmission.

So, when the data is scraped, it passes through what is called a pipeline in scrapy, this can be found in pipeline.py

```python
    import psycopg2
    from itemadapter import ItemAdapter
    
    
    class CarScraperPipeline:
        def process_item(self, item, spider):
            
            if item.get("price") is not None:
                item["price"] = int(item["price"].replace("₦", "").replace(",", 
    
            if item.get("mileage") is not None:
                item["mileage"] = int(item["mileage"].split(" ")[0])
    
            if item.get("city") is not None:
                item["city"] = item["city"].split(",")[-1].strip()
    
            if item.get("engine_size") is not None:
                item["engine_size"] = int(item["engine_size"])
    
            return item
    
    
    class PostgresNoDuplicatesPipeline:
        def __init__(self):
            self.create_connection()
    
        def create_connection(self):
            self.connection = psycopg2.connect(
                host=DB_HOST_NAME,
                database=DB_NAME,
                user=DB_USER,
                password=DB_PASSWORD,
            )
            self.cur = self.connection.cursor()
    
            self.cur.execute(
                """
                CREATE TABLE IF NOT EXISTS cars (
                    id serial PRIMARY KEY,
                    car_id VARCHAR(50) UNIQUE,
                    make VARCHAR(50),
                    model VARCHAR(50),
                    year_of_manufacture TEXT,
                    color TEXT,
                    condition TEXT,
                    mileage INT,
                    engine_size INT,
                    registered_city TEXT,
                    selling_condition TEXT,
                    bought_condition TEXT,
                    city TEXT,
                    price INT,
                    body_type TEXT,
                    fuel_type TEXT,
                    transmission TEXT
                                )
                """
            )
            self.connection.commit()
    
        def process_item(self, item, spider):
            try:
                self.store_db(item, spider)
            except psycopg2.Error as e:
                spider.logger.error(f"Error inserting data into database: {e}")
            return item
    
        def store_db(self, item, spider):
            try:
                car_id = item.get("car_id")
                assert isinstance(car_id, str), f"car_id is not a string: {car_id}"
    
                self.cur.execute(
                    """ 
                INSERT INTO cars (car_id, make, model, year_of_manufacture, color, condition,
                                      mileage, engine_size, registered_city, selling_condition, 
                                      bought_condition, city, price, body_type, fuel_type, transmission)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                    ON CONFLICT (car_id) DO NOTHING
                    """,
                    (
                        str(item.get("car_id")),
                        item.get("make"),
                        item.get("model"),
                        item.get("year_of_manufacture"),
                        item.get("color"),
                        item.get("condition"),
                        item.get("mileage"),
                        item.get("engine_size"),
                        item.get("registered_city"),
                        item.get("selling_condition"),
                        item.get("bought_condition"),
                        item.get("city"),
                        item.get("price"),
                        item.get("body_type"),
                        item.get("fuel_type"),
                        item.get("transmission"),
                    ),
                )
                self.connection.commit()
                return item
            except psycopg2.Error as e:
                self.connection.rollback()
                spider.logger.error(f"Error inserting data into database: {e}")
    
        def close_spider(self, spider):
            self.cur.close()
            self.connection.close()
```

The pipeline script does some formatting processes to the items being scrapped, then it creates a connection to a PostgreSQL and saves the items there.

A single scrape gets like 1900+ items, so I needed a bunch just to use for the machine learning. So, I had to take the whole process to the cloud.

Scrapy has a nice services [Scrapeops](https://scrapeops.io/), is a cloud-based scraping service allowing to run and scale web scraping jobs in the cloud. So, I hosted a scraper-instance on a GCP VM and connected it to scrapeops. The dashboard is below.

![scrapeops dashboard (image by author)](https://cdn-images-1.medium.com/max/3214/1*nA5yDMCvgVBc7phCIHcJrA.png)

It was running for a couple of days and was shutdown. I just needed a few data items.

## Machine Learning

![machine learning workflow (image by author)](https://cdn-images-1.medium.com/max/2000/1*yi6DWJF54NKQv4K1sepNUQ.png)

The machine learning was pretty straight forward, and I didn’t want to over-engineer anything.

```python	
    ## Importing 
    import pandas as pd
    import numpy as np
    import matplotlib.pyplot as plt
    import seaborn as sns
    
    data = pd.read_csv('data/car_data.csv')
    data.info()
    data.drop(['id', 'car_id'], axis=1, inplace=True)
    data.info()
    car_data = data.copy()
    
    ## Preprocessing
    from sklearn.compose import ColumnTransformer
    from sklearn.pipeline import Pipeline
    from sklearn.impute import SimpleImputer
    from sklearn.preprocessing import StandardScaler, OneHotEncoder, LabelEncoder
    # feature and target variables
    X = car_data.drop(columns = ["price"], axis=1)
    y = car_data["price"]
    # preprocessing numerical features
    numerical_features = ["mileage", "engine_size"]
    
    numerical_transformer = Pipeline(steps=[
        ('imputer', SimpleImputer(strategy='mean')),
        ('scaler', StandardScaler())
    ])
    
    # preprocessing categorical features
    categorical_features = ["make", "model", "color", "condition", "registered_city", "selling_condition", "bought_condition", "city", "body_type", "fuel_type", "transmission"]
    
    
    categorical_transformer = Pipeline(steps=[('imputer', SimpleImputer(strategy='most_frequent')),
    ('onehot', OneHotEncoder(handle_unknown='ignore'))])
    # combining steps
    preprocessor = ColumnTransformer(
        transformers=[
            ('num', numerical_transformer, numerical_features),
            ('cat', categorical_transformer, categorical_features)
        ])
    
    ## Model
    from sklearn.model_selection import train_test_split
    import xgboost as xgb
    from sklearn.ensemble import RandomForestRegressor
    model = Pipeline(steps=[('preprocessor', preprocessor),
                            ('regressor', RandomForestRegressor(random_state = 42))])
    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
    model.fit(X_train, y_train)
    y_pred_1 = model.predict(X_test)
    
    ## Evaluation
    from sklearn.metrics import mean_squared_error, r2_score, mean_absolute_error
    mae = mean_absolute_error(y_test, y_pred_1)
    mse = mean_squared_error(y_test, y_pred_1)
    rmse = np.sqrt(mse)
    r2 = r2_score(y_test, y_pred_1)
    
    print("Mean Absolute Error: ", mae)
    print("Mean Squared Error: ", mse)
    print("Root Mean Squared Error: ", rmse)
    print("R2 Score: ", r2)
    
    ## Tuning
    from sklearn.model_selection import GridSearchCV
    param_grid = {
        'regressor__n_estimators' : [100, 200, 300],
        'regressor__max_depth' : [None, 10, 20, 30],
    }
    grid_search = GridSearchCV(model, param_grid, cv=5, n_jobs=-1, verbose=1, scoring="neg_mean_absolute_error")
    grid_search.fit(X_train, y_train)
    best_params = grid_search.best_params_
    best_params
    y_pred_2 = grid_search.predict(X_test)
    
    mae = mean_absolute_error(y_test, y_pred_2)
    mse = mean_squared_error(y_test, y_pred_2)
    rmse = np.sqrt(mse)
    r2 = r2_score(y_test, y_pred_2)
    
    print("Mean Absolute Error: ", mae)
    print("Mean Squared Error: ", mse)
    print("Root Mean Squared Error: ", rmse)
    print("R2 Score: ", r2)
    
    ## Saving model
    import joblib
    joblib.dump(grid_search, "model/car_prediction_model.pickle") 

```

I loaded the data from the PostgreSQL server into a csv, can be found in import.ipynb , preprocess the data by using sklearn column transformer and simple imputer and pass through sklearn pipeline. After some preprocessing, I used Random Forest Regressor to train the model and also did some parameter tuning and finally saved the best model in a pickle file. All these can be found in main.ipynb

## Deployment

With the model’s pickle file, I decided to make both an API and a UI differently using Fast API and gradio respectively.

For the Fast API, I created a `model.py`

```python
    import pandas as pd
    import joblib
    
    from pydantic import BaseModel
    from sklearn.compose import ColumnTransformer
    from sklearn.pipeline import Pipeline
    from sklearn.impute import SimpleImputer
    from sklearn.preprocessing import StandardScaler, OneHotEncoder
    from sklearn.ensemble import RandomForestRegressor
    
    class CarPrices(BaseModel):
        make: str
        model: str
        year_of_manufacture: int
        color: str
        condition: str
        mileage: float
        engine_size: float
        registered_city: str
        selling_condition: str
        bought_condition: str
        city: str
        body_type: str
        fuel_type: str  
        transmission: str
    
    class CarPriceModel:
        """
        model to predict the prices of cars from car45.com
        """
    
        def __init__(self):
            self.data = pd.read_csv("data/car_data.csv")
            self.model_fpath_ = "model/car_prediction_model.pickle"
    
            try:
                self.model = joblib.load(self.model_fpath_)
            except Exception:
                self.model = self.train_model()
                joblib.dump(self.model, self.model_fpath_)
    
        def preprocessor(self):
            numerical_features = ["mileage", "engine_size"]
            categorical_features = [
                "make",
                "model",
                "color",
                "condition",
                "registered_city",
                "selling_condition",
                "bought_condition",
                "city",
                "body_type",
                "fuel_type",
                "transmission",
            ]
    
            numerical_transformer = Pipeline(
                steps=[
                    ("imputer", SimpleImputer(strategy="mean")),
                    ("scaler", StandardScaler()),
                ]
            )
    
            categorical_transformer = Pipeline(
                steps=[
                    ("imputer", SimpleImputer(strategy="most_frequent")),
                    ("onehot", OneHotEncoder(handle_unknown="ignore")),
                ]
            )
    
            preprocessor = ColumnTransformer(
                transformers=[
                    ("num", numerical_transformer, numerical_features),
                    ("cat", categorical_transformer, categorical_features),
                ]
            )
    
            return preprocessor
    
        def train_model(self):
            preprocessor = self.preprocessor()
            X = self.data.drop("price", axis=1)
            y = self.data["price"]
    
            model = Pipeline(
                steps=[
                    ("preprocessor", preprocessor),
                    ("model", RandomForestRegressor(random_state=42)),
                ]
            )
    
            model.fit(X, y)
    
            return model
        
        def predict(self, data):
            data = pd.DataFrame([data])
            preprocessor = self.model.named_steps['preprocessor']
            X_processed = preprocessor.transform(data)
            prediction = self.model.named_steps['model'].predict(X_processed)
    
            return prediction[0]
```

In the script it loads up the csv file using pandas, uses the same preprocessing methods that was used to train the model and a predict method. and the main.py` file to run the API.

```python
    import uvicorn
    from fastapi import FastAPI
    from model import CarPrices, CarPriceModel
    
    app = FastAPI()
    model = CarPriceModel() 
    
    @app.get("/")
    def home():
        return { 'Message' : 'Welcome to the Car Price Prediction API' }
    
    @app.post("/predict")
    async def predict_price(car: CarPrices):
        data = car.model_dump()
        prediction = model.predict(data)
    
        return {"price": prediction}
    
    if __name__ == "__main__":
        uvicorn.run(app, host="127.0.0.1", port = 8000)
```

For the gradio, first of all, gradio is one of the fastest way to demo machine learning models with a beautiful user interface. Not to bore you with the detail explanation of the way it works, but the code for it is below, which can be found in gradio_app.py`

```python
    import pandas as pd
    import gradio as gr
    import warnings
    from model import CarPriceModel, CarPrices
    
    warnings.filterwarnings("ignore", category=UserWarning, module="sklearn")
    
    data = pd.read_csv("data/car_data.csv")
    
    unique_values = {
        "make": list(data["make"].unique()),
        "model": list(data["model"].unique()),
        "color": list(data["color"].unique()),
        "condition": list(data["condition"].unique()),
        "registered_city": list(data["registered_city"].unique()),
        "selling_condition": list(data["selling_condition"].unique()),
        "bought_condition": list(data["bought_condition"].unique()),
        "city": list(data["city"].unique()),
        "body_type": list(data["body_type"].unique()),
        "fuel_type": list(data["fuel_type"].unique()),
        "transmission": list(data["transmission"].unique()),
    }
    
    
    def predict_car_price(
        make,
        model,
        year_of_manufacture,
        color,
        condition,
        mileage,
        engine_size,
        registered_city,
        selling_condition,
        bought_condition,
        city,
        body_type,
        fuel_type,
        transmission,
    ):
        car = CarPrices(
            make=make,
            model=model,
            year_of_manufacture=int(year_of_manufacture),
            color=color,
            condition=condition,
            mileage=float(mileage),
            engine_size=float(engine_size),
            registered_city=registered_city,
            selling_condition=selling_condition,
            bought_condition=bought_condition,
            city=city,
            body_type=body_type,
            fuel_type=fuel_type,
            transmission=transmission,
        )
    
        car_model = CarPriceModel()
    
        prediction = car_model.predict(car.model_dump())
        formatted_prediction = f"₦{prediction:,.2f}"
        return formatted_prediction
    
    
    with gr.Blocks() as interface:
        gr.Markdown(
            "# Car Price Prediction\nPredict the price of a car based on its features."
        )
    
        with gr.Tab("Basic Information"):
            make = gr.Dropdown(label="Make", choices=unique_values["make"])
            model = gr.Dropdown(label="Model", choices=unique_values["model"])
            year_of_manufacture = gr.Number(label="Year of Manufacture")
            color = gr.Dropdown(label="Color", choices=unique_values["color"])
            condition = gr.Dropdown(label="Condition", choices=unique_values["condition"])
    
        with gr.Tab("Sepcifications"):
            mileage = gr.Number(label="Mileage")
            engine_size = gr.Number(label="Engine Size")
    
        with gr.Tab("Location Information"):
            registered_city = gr.Dropdown(
                label="Registered City", choices=unique_values["registered_city"]
            )
            city = gr.Dropdown(label="City", choices=unique_values["city"])
    
        with gr.Tab("Condition and Type"):
            selling_condition = gr.Dropdown(
                label="Selling Condition", choices=unique_values["selling_condition"]
            )
            bought_condition = gr.Dropdown(
                label="Bought Condition", choices=unique_values["bought_condition"]
            )
            body_type = gr.Dropdown(label="Body Type", choices=unique_values["body_type"])
            fuel_type = gr.Dropdown(label="Fuel Type", choices=unique_values["fuel_type"])
            transmission = gr.Dropdown(
                label="Transmission", choices=unique_values["transmission"]
            )
    
        predict_button = gr.Button("Predict Price")
        predicted_price = gr.Textbox(label="Predicted Price")
    
        predict_button.click(
            fn=predict_car_price,
            inputs=[
                make,
                model,
                year_of_manufacture,
                color,
                condition,
                mileage,
                engine_size,
                registered_city,
                selling_condition,
                bought_condition,
                city,
                body_type,
                fuel_type,
                transmission,
            ],
            outputs=predicted_price,
        )
    
    interface.launch(server_name="0.0.0.0", server_port=7860)
```

The demo for the gradio UI is in the GitHub readme: [https://github.com/Duks31/car_price-prediction](https://github.com/Duks31/car_price-prediction)

Then after testing the API and the UI, I dockerised it to make it available regardless of the dev environment that you are run it from. The tricky thing that I had to do here is to run the fast API and the gradio UI from the same image running on 2 different ports.

So, I learnt about this software [supervisord](http://supervisord.org/). Supervisor is a client/server system that allows its users to monitor and control a number of processes on UNIX-like operating systems.

my supervisord.conf`file looked like this

```toml

    [supervisord]
    nodaemon=true
    
    [program:fastapi]
    command=uvicorn main:app --host 0.0.0.0 --port 8000
    autostart=true
    autorestart=true
    stderr_logfile=/var/log/fastapi.err.log
    stdout_logfile=/var/log/fastapi.out.log
    
    [program:gradio]
    command=python gradio_app.py
    autostart=true
    autorestart=true
    stderr_logfile=/var/log/gradio.err.log
    stdout_logfile=/var/log/gradio.out.log
```

This configuration file sets up two programs to be managed by supervisord: a FastAPI application and a Gradio application. Both applications are configured to start automatically when supervisord starts and to restart if they crash. Log files are specified for each application's standard output and error, allowing you to monitor their activity and troubleshoot issues.

So, with that setup I the created the Dockerfile :

```Bash
    # syntax=docker/dockerfile:1
    
    ARG PYTHON_VERSION=3.11.9
    FROM python:${PYTHON_VERSION} as base
    
    ENV PYTHONDONTWRITEBYTECODE=1
    
    ENV PYTHONUNBUFFERED=1
    
    WORKDIR /app
    
    COPY requirements.txt .
    
    RUN pip install --no-cache-dir -r requirements.txt
    
    RUN apt-get update && apt-get install -y supervisor && rm -rf /var/lib/apt/lists/*
    
    COPY supervisord.conf /etc/supervisor/supervisord.conf
    
    COPY . .
    
    EXPOSE 8000
    
    EXPOSE 7860
    
    CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
```

This Dockerfile builds a Docker image with a Python environment, installs the necessary Python dependencies, sets up supervisor to manage the FastAPI and Gradio applications, and prepares the container to expose the necessary ports for these applications. The use of supervisord ensures that both applications are started and monitored within the container, facilitating efficient process management.

Finally, I built the image using docker compose and pushed it to Docker hub: [chidubem/car_pred-app general (docker.com)](https://hub.docker.com/repository/docker/chidubem/car_pred-app/general)

**CI/CD (*somewhat)***

Thereafter, I needed to automate the process, making it such that when there is a PR (pull-request) to the main branch in my repo, the PR may be an update to the model to improve its accuracy. code below:

```yaml
    name: Build and Push Docker Image
    
    on:
      # push:
      #   branches:
      #     - main
          
      pull_request:
        types: [closed]
        branches:
          - main
          
    jobs:
      build-and-push:
        if: github.event.pull_request.merged == true
        runs-on: ubuntu-latest
    
        steps:
        - name: Checkout repository
          uses: actions/checkout@v4
    
        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v2
    
        - name: Log in to Docker Hub
          uses: docker/login-action@v2
          with:
            username: ${{ secrets.DOCKER_USERNAME }}
            password: ${{ secrets.DOCKER_PASSWORD }}
        
        - name: Set up tag for the Docker image
          id: vars
          run: echo "TAG=$(date +%s)" >> $GITHUB_ENV
    
        - name: Build the Docker image
          run: docker build . --file Dockerfile --tag chidubem/car_pred-app:${{ env.TAG }}
    
        - name: Push the Docker image
          run: docker push chidubem/car_pred-app:${{ env.TAG }}
    
        - name: Log out from Docker Hub
          run: docker logout
```

I think it is more of a CD (continuous deployment) process, this GitHub Actions workflow automates the process of building and pushing a Docker image to Docker Hub as part of a Continuous Deployment (CD) pipeline. It is triggered when a pull request to the main branch is closed and successfully merged. The workflow runs on an Ubuntu environment and performs several steps: checking out the repository, setting up Docker Buildx for advanced build capabilities, and logging into Docker Hub using credentials stored in GitHub Secrets. It assigns a unique tag to the Docker image based on the current Unix timestamp, builds the Docker image using the Dockerfile in the repository, and pushes the tagged image to Docker Hub. Finally, it logs out from Docker Hub to maintain security. This setup ensures that every change merged into the main branch results in a new version of the Docker image being deployed to Docker Hub, facilitating continuous deployment.

