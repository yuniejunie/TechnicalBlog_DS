## Getting Started with Data lineage and Metadata automation

This project shows how to map the databases based on their dependencies and provide its data lineage. 

## Table of Contents<a name="top"></a>
* [Situation and Complications](#Situation_and_Complications)
* [Solution](#Solution)
* [Limitations](#limitations)
* [Maintenance guides](#maintenance)

### Situation and Complications<a name="Situation_and_Complications"></a>
This project is to have **data traceability** that goes all the way back to the original data source.
Views in the organization are often built off another view that was built off the other views, which decrease its traceability toward its sources. 
Furthermore, this practice is currently being done manually, which is time-consuming and prone to errors. 
Another challenge is to present this information to data consumers in a useful manner.  
[Top↑](#top) 

### Solution<a name="Solution"></a>
I chose [the Graph theory](https://en.wikipedia.org/wiki/Graph_theory#Computer_science) as a solution for those complications.
In short, the model will connect each tables based on their relationships. 
Each node represents a table/view and each edge(connecting line) represents their relationship. 
The direction shows which way this table is being sourced toward.
For example, Table B is formed based on 'SELECT col1, col2 FROM TableA'. In this case, the graph will show up like this.

![Nodes and edge](https://github.com/yuniejunie/Cgl_DataLineage/blob/master/readme_reference/nodes_edge.png)

As this process iterates within tables in each DB, the Graph will be able to show the data lineage as the more node-node connections are added.

![Nodes_to_node_iteration](https://github.com/yuniejunie/Cgl_DataLineage/blob/master/readme_reference/data_lineage_principle.PNG)

If you take a look at the picture above, Table B's data lineage could be drawn as two iterations happened around Table B; 
Now we can say Table B is based off of Table A and it is a source of Table C.

If we plot all tables/views in this way, it would be helpful to understand the flow of data and to build the data lineage along with its flow.

As a result, the following graph shows how the tables are mapped visually. From here, "The graph" will refer the same graph shown below in the backend.  

####### Graph without legend should go in here

Data lineage model works with automated metadata ingestion model as well since they both use the graph based on the Graph theory.
It is written in **Python 3**.

[Top↑](#top) 

## Limitations<a name="limitations"></a>

There are several limitations existing in the current approach. 

1. Currently Column comments for VIEWS are not realized in codes due to Cloudera Impala/Hive's technical limitations. 
Column comments for views are only able to be added on CREATE statements for each views. 
    - Contacted Cloudera and opened a feature request for this. 
2. Other softwares in the industry might be easier to deploy
    - I built this application in-house, but there are several graph database softwares exist in the market(e.g. Neo4j). 
      The reasons why I ended up building this application from the scratch were the following:
        1. Customizable: My client uses Cloudera Impala/Hive and it has the technical limitations. 
        I found out that metadata from its source tables were not inherited down to the predecessors, which results in lots of empty metadata. 
        My client was not interested in using any 3rd party application to look up data lineage or metadata. 
        Therefore, I had to find a way of ingesting metadata back to their database directly and decided to build a script from the scratch.
3. It is possible that the data lineage is missing some connections.
    
    - This application is based on each view's DDL statement. 
    It works great if Team Phoenix has a full access to all DDLs of the DB that's being mapped.
    If we don't have a full access, it is impossible to map that relationship. 
    Therefore, there is a potential concern that this application is missing some relationship 
    such as a view in other DBs was built upon one of our tables/views while we don't have any access to their DBs.
    
    ![No_Access_DBs_Limitation](https://github.com/yuniejunie/Cgl_DataLineage/blob/master/readme_reference/no_access_views.PNG)

## Maintenance guides<a name="maintenance"></a>
This section provides helpful information for someone to do maintenance. 

1. **connector.py**: To set up the connection to Impala and to build a graph database.
    - class **connector()** 
        - **connector(self, imp_host, dbname)**: To establish the Impala connection and return its cursor.
        - **close_conn(self, cursor)**: To close cursor when the job done. 
        - **get_tables(self, cursor)**: To get the list of tables of the selected Database.
        - **get_columns(self, cursor, table)**: To get the list of columns of the selected table.
        - **get_create_statements(self, tables, cursor, DB, show_failures = False)**: To get the create statement to analyze/extract the data lineage. `show_failures = TRUE` can be used to return the list of tables not process at this step.     
        - **parse_sql(self, sql)**: To parse DDL statement by using `sqlparse`
        - **build_graph(self, cursor, DB, G = nx.DiGraph())**: To build a graph database. The default is to build it from the scratch. 
            You can build a new graph over other existing graph by using `G = [EXISTING_G]`. It will be the best to use when you want to combine the graph in Drona and Peanut.
        - **update_G(self, cursor, DB, G)**: To refine the graph by deleting any nodes not found in the current DBs.  

2. **table_attributes.py**: To add table attributes to each node in the graph database(G)
    - **get_table_properties(table, cursor)**: To get table properties from DBs. Its return type is Dict.
        Currently it tracks COMMENT, SOURCE, LOAD_FREQUENCY, SECURITY_CLASSIFICATION, and CONTACT_INFO.
    - **get_col_comments(table, cursor, G, DB)**: To get column comments from DBs. Its return type is Dict.
    - **generate_metadata_metrics(table_props)**: To return the progress in table property. 
        It accepts the outcome of `get_table_properties` as a parameter. Its return type is a dict.
    - **generate_column_comment_metrics(col_comments)**: To return the progress of column comments.
        It accepts the outcome of `get_col_comments` as a parameter. Its return type is a dict.
    - **get_meta_data(G, tables, DB, cursor)**: To return the table_attributes to add on each node.
        This function uses every other functions mentioned above to get the final outcome. 
        It also iterates to enable metadata inheritance between sources and target tables.
    - **prep(G, tables, DB, cursor)**: To prepare ingesting table attributes to nodes.
        Mostly it combines table attributes to input to each node and it also changes the format of each node names for the consistency.
    - **update_G_metadata(G, tables, DB, cursor, tables_lists)**: To add table attributes to each node in G.
        Its return is a list of unfinished/unprocessed tables and its type is a list.
    - **get_outward_cols(G, table)**:
    - **get_outward_tables(G, table)**: To return a list of tables that uses the selected table as a source.
    - **get_tables_from_source(G, DB)**: To retrieve tables that use the tables in the input database as a source.
        Its return type is a Pandas DataFrame, which can easily be converted to .csv format.
    - **get_inward_cols(G, table)**: 
    - **get_inward_tables(G, table)**: To return a list of tables this table is using as sources.
    - **get_inward_tables_from_source_DB(G, DB)**: To retrieve tables that tables in the input database uses as sources.
        Its return type is a Pandas DataFrame.
    - **get_tables_from_source_table(G, table)**: To retrieve tables that use the selected table as a source.
        Its return type is a Pandas DataFrame.
    - **get_inward_tables_from_source_table(G, table)**: To retrieve tables that the selected table uses as sources.
        Its return type is a Pandas DataFrame.
    - **get_col_comments_from_source(G, cursor, table, DB)**: To inherit column comments from its sources and the mirror table in other environments. (e.g. dev_internal_fair & qas_internal_fair & prd_internal_fair)
        Its return type is a dict. This function is being used in `get_meta_data()`.
    - **check_table_type(DB)**: To get the list of tables and views. It returns 2 outputs: a list of views, a list of tables.
    - **get_metadata_status(G, input_tpe, userinput)**: To check the metadata progress. input_type can be specified either 1 or 2.
        1 is for DB input and 2 is for table level input. It calculates its metadata progress in percentage.
        Its return type is a Pandas DataFrame.
        
