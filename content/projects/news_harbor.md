---
title: "News Harbor"
excerpt: "News Harbor is a website that detects and shows the sentiment of News Articles <br/><img src='/images/newsharbor.png'>"
collection: portfolio
date: 2023-12-23
---

Check out the project on [github](https://github.com/Duks31/news-harbor)

![news harbor](/images/newsharbor.png)

## Overview 

News Harbor is a news platform that detects the sentiment of News articles. check out this [demo](https://drive.google.com/file/d/1FEtVAbNGc0cyHIeUU4fDTx3miG0SN6r4/view?usp=sharing)

This is how the project workflow is: ![workflow](/images/news_harbor_workflow.png)

Sentiment analysis in NLP is the process of determining the emotion or tone behind a piece of text, classifying it as positive, negative, or neutral. It's used to gauge public opinion, feedback, or reactions. Techniques include lexicon-based methods using sentiment dictionaries and machine learning models trained on labeled data. In this project it was used to classifiy the news articles. 

## Usage 

### How it works 

- The user enters the news_article API_key; in this case i used my api key in the `.env` file in secrets folder which was loaded to the `news_api_data.py` file.
- The API is then used to fetch the news article from the news api, which returns a JSON. chcek the `news_api_data.py` file and the `news_api_data.json` file in core folder and data folder respectively.
- The JSON is then converted to a pandas dataframe and then the dataframe is sent to the sentiment analysis model. check the `convert.py` file in core folder. 
- The sentiment analysis model returns the sentiment of the news article in a CSV file format. check the `model.ipynb` file in model folder. Note: Google Colab was used in running the sentiment analysis model.
- The sentiment in CSV is sent to the frontend.
- The frontend displays the sentiment of the news article. check the `main.py` file in the frontend folder. 


### Tech Used
* API: [News API](https://newsapi.org/) was used for the news feed Getting the news data for the model
* Hugging Face sentiment analysis model: [Hugging Face](https://huggingface.co/) was used for the sentiment analysis model
* Streamlit was used for the frontend of the project. [Streamlit](https://streamlit.io/)

