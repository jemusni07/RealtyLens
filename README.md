**Note:** The Snowflake connection is currently not working and is being transitioned. 


# RealtyLens

![RealtyLens](./images/realtylens.png)


RealtyLens is an application that provides analytic lens for daily rent and sale listings. This is my capstone submission for Zach Wilson's DataExpert Data Engineering bootcamp.

**Capabilities**
- Sales and Rent Market Overview - The Market Dashboard provides averages on a given aggregate levels of bedroom and property type.  

- Market Lifecycle - Realtylens tracks the lifecycle of the rent market. Day to day, it determines whethere a property listing is new, retained, churned, resurrected and inactive. 

- Market Trends - Checks the price and activity trends of properties over time.

- Investment Opportunities - THe project provides an estimated rent price for active sale listings. This gives an investor an insight on whether a property is overpriced or underpriced, cash flow positive or negative. It also computes the possible annual yield if the property is bought at the given price and rented out at the estimated rent price. The rent estimation is powered by a machine learning model.

You can check out the app here - https://realtylens.streamlit.app/
As of now, the data pipeline and dashboard only includes data for the city of Philadelphia. Please reach out to me if you have any questions or feedback. 

## Data Sources
**Rentcast API** - https://www.rentcast.io/

- Philadelphia Active Rent and Sale Listings (*JSON*) - https://developers.rentcast.io/reference/property-listings-schema
 
 These are active Listings extracted daily to what is on the market for rent and sale. Capturing the snapshot of the market daily also means day to day some listings are removed and some listings are added. When the listing is removed, it could mean anything but it is also could be likely sold or rented out. It is then possible to see the market trends and lifecycle.

- Philadelphia Property Details (*JSON*) - https://developers.rentcast.io/reference/property-data-schema

The daily active listings from Rentcast API unfortunately does not have details like bathroom and bedroom counts so I had to extract the details separately from another rentcast api endpoint. I ended up extracting almost 700k property records for Philadelphia that serves as a lookup table for the daily active listings to build the property dimension table.

**Open Data Philly** - https://opendataphilly.org/datasets/

- Zoning Base Districts (*GeoJSON*) - https://opendataphilly.org/datasets/zoning-base-districts/

I found out that not all properties from rentcast api have a zoning dimensions so I had to extract from Open Data Philly to get the latest zoning districts for Philadelphia. The data pipeline will do a spatial join if the property's coordinates is within a zoning districts polygon coordinates.

- Historic Landmarks (*GeoJSON*) - https://opendataphilly.org/datasets/city-landmarks/

For future enhancement and analysis, we can do spatial joins to determine whether what landmarks are within a property's vicinity.

- Zip code data (*GeoJSON*) - https://opendataphilly.org/datasets/zip-codes/

Also will be useful for future enhancement, we can visualize the zip code borders on the map dashboard.

## Data Pipeline and Architecture

![Data Pipeline](./images/realtylens_conceptual_diagram.png)

**Data Orchestration**

The data pipeline is orchestrated by Astronomer managed Airflow

![Data Orchestration](./images/airflow_1.png)

*These are the DAGs that are deployed to Astronomer managed Airflow*

![Data Orchestration](./images/airflow_2.png)

*Daily pipeline DAG for Extract Load Transform (ELT) on the active listings, the runs before the last three days captures the development stages of the project*

- Daily DAG - This pipeline runs daily to extract the latest active listings enriched with property details stored in Snowflake, transformed by DBT and served to the streamlit dashboard. The model also runs rent prediction for active sale listings.
- Weekly DAG - This pipeline runs weekly to extract the latest zoning districts and zip codes stored in Snowflake, transformed by DBT and served to the streamlit dashboard
- ML DAG - triggerd manually to pull out from the latest feature store and run the rent prediction machine learning training, the model is then store to a registry

**Data Storage and Warehouse**

The data warehouse is powered by S3 and Snowflake.
- AWS S3 - the extracted data from the sources is stored in S3 partitioned by it source, state, city, and date.
- Snowflake - the data is staged, stored and transformed using Snowflake.

**Transformation and Data Quality**

Data Build Tool(DBT) transforms the raw data into facts, dimensions and analytics tables. IT also provides a data quality framework to ensure the data is clean and consistent. Also DBT helps building the feature store for the machine learning model.

**Data Modeling and Cumulative Tables**

Data is modeled into facts and dimensions in a star schema to capture the daily active listings. The cunulative tables are built by doing full outer joins with the previous day's data and then tracks price changes and the activity of a listing over time.

![Data Lineage](./images/data_lineage.png)

- (fct_sale_listing, fct_rent_listing) - these are the facts tables that capture the daily active listings.

- (dim_property, dim_mls, dim_location) - these are the tables that provides dimensions for the facts tables.

- (cumulative_sale_listing, cumulative_rent_listing) - these tables tracks the state and price changes of a listing every the date of its listing

- (sale_market_and_seasonality, sale_market_elasticity_market_impact, sale_price_marlet_analysis, rent_price_market_analysis, rent_price_optimization, rent_market_health_index, rent_lifecycle) - these tables form the data marts that serves the analytic dashboard.

- (rent_ml_feature_store, predicted_rent_price) - the feature store is derived from fct_rent_listing and dim_property tables. The feature store table provides a map columns for both categorical and numerical columns.
he predicted rent price table are the ones generated by the machine learning model for estimating rent prices for active sale listings.

**Machine Learning**

The project the highlight the whole data engineering pipeline and machine learning capability is a feature of this pipeline. With this, the machine learning pipeline that was integrated into the project is a baseline model that can be improved and enhanced in the future. A gradient boosting regression model was used to predict the rent price for active sale listings.

**Backfilling**
The project started to extract data on February 4, 2025. As the daily raw files are sent to S3, there is capability to drop tables and backfill during the development phase. This is especially useful for the cumulative tables that tracks the state and price changes of a listing day to day.

**Streamlit Analytics Dashboard and Property Lookup**
As shown above, the streamlit application is capable of showing a property in a map view and provide market statistics and insights for the given filters. In another view it also provide dashboard for the market analysis, trends and lifecycle.

## Future Enhancements

**PostgresSQL** The project is currently using Snowflake for row level filters and lookups. Snowflake is designed for analtytic workloads. For future enhancements, the project can be enhanced to use PostgresSQL for the transactional and lookup workloads while Snowflake can be used only for analytics workloads.

**Streamlit Dashboard** The application is using streamlit cloud's free tier and shared memory. It is capable of rendering all properties in a map but very slow. The project needs more memory and compute for a better UI experience. The market analytics dashboard also needs enhancement to guide the user on how to read the visualizations.

**Machine Learning** The model is using a baseline regression model and as it sits righ now, it gives an .64 R2 score which can be improved by optimization, fine tuning and feature engineering.