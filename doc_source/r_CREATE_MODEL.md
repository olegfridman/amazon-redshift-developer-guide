# CREATE MODEL<a name="r_CREATE_MODEL"></a>


|  | 
| --- |
| This is prerelease documentation for the machine learning feature for Amazon Redshift, which is in preview release\. The documentation and the feature are both subject to change\. We recommend that you use this feature only with test clusters, and not in production environments\. For preview terms and conditions, see Beta Service Participation in [AWS Service Terms](https://aws.amazon.com/service-terms/)\.   | 

The CREATE MODEL statement offers flexibility in the number of parameters used to create the model\. Depending on their needs or problem type, users can choose their preferred preprocessors, algorithms, problem types, or hyperparameters\.

Before you use the CREATE MODEL statement, complete the prerequisites in [Cluster setup for using Amazon Redshift ML](cluster-setup.md)\. The following is a high\-level summary of the prerequisites\.
+ Create an Amazon Redshift cluster with the AWS console or the AWS Command Line Interface \(AWS CLI\)\.
+ Attach the AWS Identity and Access Management \(IAM\) policy while creating the cluster\.
+ To allow Amazon Redshift and SageMaker to assume the role to interact with other services, add the appropriate trust policy to the IAM role\.

For details for the IAM role, trust policy, and other prerequisites, see [Cluster setup for using Amazon Redshift ML](cluster-setup.md)\.

Following, you can find different use cases for the CREATE MODEL statement\.
+ [Simple CREATE MODEL](#r_simple_create_model)
+ [CREATE MODEL with user guidance](#r_user_guidance_create_model)
+ [CREATE XGBoost models with AUTO OFF](#r_auto_off_create_model)
+ [Bring your own model \(BYOM\)](#r_byom_create_model)
+ [Full CREATE MODEL](#r_full_create_model)

## Simple CREATE MODEL<a name="r_simple_create_model"></a>

The following summarizes the basic options of the CREATE MODEL syntax\.

### Simple CREATE MODEL syntax<a name="r_simple-create-model-synposis"></a>

```
CREATE MODEL model_name 
FROM { table_name | ( select_query ) }
TARGET column_name
FUNCTION prediction_function_name
IAM_ROLE 'iam_role_arn'
SETTINGS (
  S3_BUCKET 'bucket',
  [ MAX_CELLS integer ]
)
```

### Simple CREATE MODEL parameters<a name="r_simple-create-model-parameters"></a>

 *model\_name*   
The name of the model\. The model name in a schema must be unique\.

FROM \{ *table\_name* \| \( *select\_query* \) \}  
The table\_name or the query that specifies the training data\. They can either be an existing table in the system, or an Amazon Redshift\-compatible SELECT query enclosed with parentheses, that is \(\)\. There must be at least two columns in the query result\. They must be only strings and numerics\. 

TARGET *column\_name*  
The name of the column that becomes the prediction target\. The column must exist in the FROM clause\. 

FUNCTION *prediction\_function\_name*   
A value that specifies the name of the Amazon Redshift machine learning function to be generated by the CREATE MODEL and used to make predictions using this model\. The function is created in the same schema as the model object and can be overloaded\.  
Amazon Redshift machine learning supports models, such as Xtreme Gradient Boosted tree \(XGBoost\) models for regression and classification\.

IAM\_ROLE *'iam\_role\_arn'*  
The Amazon Resource Name \(ARN\) for an AWS Identity and Access Management \(IAM\) role that your cluster uses for authentication and authorization\. As a minimum, the IAM role must have permission to perform a LIST operation on the Amazon S3 bucket that is used for unloading training data and staging of Amazon SageMaker artifacts\. The following shows the syntax for the IAM\_ROLE parameter string for a single ARN\.  

```
IAM_ROLE 'arn:aws:iam::aws-account-id:role/role-name'
```

 *S3\_BUCKET *'bucket'**   
The name of the Amazon S3 bucket that you previously created used to share training data and artifacts between Amazon Redshift and SageMaker\. Amazon Redshift creates a subfolder in this bucket prior to unload of the training data\. When training is complete, Amazon Redshift deletes the created subfolder and its contents\. 

MAX\_CELLS integer   
The maximum number of cells to export from the FROM clause\. The default is 1,000,000\.   
The number of cells is the product of the number of rows in the training data \(produced by the FROM clause table or query\) times the number of columns\. If the number of cells in the training data are more than that specified by the max\_cells parameter, CREATE MODEL downsamples the FROM clause training data to reduce the size of the training set below MAX\_CELLS\. Allowing larger training datasets can produce higher accuracy but also can mean the model takes longer to train and costs more\.  
For information about costs of using Amazon Redshift, see [Costs for using Amazon Redshift ML](cost.md)\.  
For more information about costs associated with various cell numbers and free trial details, see [Amazon Redshift pricing](https://aws.amazon.com/redshift/pricing)\.

## CREATE MODEL with user guidance<a name="r_user_guidance_create_model"></a>

Following, you can find a description of options for CREATE MODEL in addition to the options described in [Simple CREATE MODEL](#r_simple_create_model)\.

By default, CREATE MODEL searches for the best combination of preprocessing and model for your specific dataset\. You might want additional control or introduce additional domain knowledge \(such as problem type or objective\) over your model\. In a customer churn scenario, if the outcome “customer is not active” is rare, then the F1 objective is often preferred to the accuracy objective\. Because high accuracy models might predict “customer is active” all the time, this results in high accuracy but little business value\. For information about F1 objective, see [AutoMLJobObjective](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_AutoMLJobObjective.html) in the *Amazon SageMaker API Reference*\.

Then the CREATE MODEL follows your suggestions on the specified aspects, such as the objective\. At the same time, the CREATE MODEL automatically discovers the best preprocessors and the best hyperparameters\. 

### CREATE MODEL with user guidance syntax<a name="r_user_guidance-create-model-synposis"></a>

CREATE MODEL offers more flexibility on the aspects that you can specify and the aspects that Amazon Redshift automatically discovers\.

```
CREATE MODEL model-name 
FROM { table_name | ( select_statement ) }
TARGET column_name
FUNCTION function_name
IAM_ROLE 'iam_role_arn'
[ PROBLEM_TYPE ( REGRESSION | BINARY_CLASSIFICATION | MULTICLASS_CLASSIFICATION ) ]
[ OBJECTIVE ( 'MSE' | 'Accuracy' | 'F1' | 'F1Macro' | 'AUC') ]
SETTINGS (
  S3_BUCKET 'bucket', |
  S3_GARBAGE_COLLECT { ON | OFF }, |
  KMS_KEY_ID 'kms_key_id', |
  MAX_CELLS integer, |
  MAX_RUNTIME integer (, ...)
)
```

### CREATE MODEL with user guidance parameters<a name="r_user_guidance-create-model-parameters"></a>

 *PROBLEM\_TYPE \( REGRESSION \| BINARY\_CLASSIFICATION \| MULTICLASS\_CLASSIFICATION \)*   
\(Optional\) Specifies the problem type\. If you know the problem type, you can restrict Amazon Redshift to only search of the best model of that specific model type\. If you don't specify this parameter, a problem type is discovered during the training, based on your data\.

OBJECTIVE \( 'MSE' \| 'Accuracy' \| 'F1' \| 'F1Macro' \| 'AUC'\)  
\(Optional\) Specifies the name of the objective metric used to measure the predictive quality of a machine learning system\. This metric is optimized during training to provide the best estimate for model parameter values from data\. If you don't specify a metric explicitly, the default behavior is to automatically use MSE: for regression, F1: for binary classification, Accuracy: for multiclass classification\. For more information about objectives, see [AutoMLJobObjective](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_AutoMLJobObjective.html) in the *Amazon SageMaker API Reference*\.

MAX\_CELLS integer   
\(Optional\) Specifies the number of cells in the training data\. This value is the product of the number of records \(in the training query or table\) times the number of columns\. The default is 1,000,000\.

MAX\_RUNTIME integer   
\(Optional\) Specifies the maximum amount of time to train\. Training jobs often complete sooner depending on dataset size\. This specifies the maximum amount of time the training should take\. The default is 5,400 \(90 minutes\)\.

S3\_GARBAGE\_COLLECT \{ ON \| OFF \}  
\(Optional\) Specifies whether Amazon Redshift performs garbage collection on the resulting datasets used to train models and the models\. If set to OFF, the resulting datasets used to train models and the models remains in Amazon S3 and can be used for other purposes\. If set to ON, Amazon Redshift deletes the artifacts in Amazon S3 after the training completes\. The default is ON\.

KMS\_KEY\_ID 'kms\_key\_id'  
\(Optional\) Specifies if Amazon Redshift uses server\-side encryption with an AWS KMS key to protect data at rest\. Data in transit is protected with Secure Sockets Layer \(SSL\)\. 

 *PREPROCESSORS 'string' *   
\(Optional\) Specifies certain combinations of preprocessors to certain sets of columns\. The format is a list of columnSets, and the appropriate transforms to be applied to each set of columns\. Amazon Redshift applies all the transformers in a specific transformers list to all columns in the corresponding ColumnSet\. For example, to apply OneHotEncoder with Imputer to columns t1 and t2, use the sample command following\.  

```
CREATE MODEL customer_churn
FROM customer_data 
TARGET 'Churn'
FUNCTION predict_churn
IAM_ROLE 'iam_role'
PROBLEM_TYPE BINARY_CLASSIFICATION
OBJECTIVE 'F1'
PREPROCESSORS '[
...
  {"ColumnSet": [
      "t1",
      "t2"
    ],
    "Transformers": [
      "OneHotEncoder",
      "Imputer"
    ]
  },
  {"ColumnSet": [
      "t3"
    ],
    "Transformers": [
      "OneHotEncoder"
    ]
  },
  {"ColumnSet": [
      "temp"
    ],
    "Transformers": [
      "Imputer",
      "NumericPassthrough"
    ]
  }
]'
SETTINGS (
  S3_BUCKET 'bucket'
)
```

Amazon Redshift supports the following transformers:
+ OneHotEncoder — Typically used to encode a discrete value into a binary vector with one non\-zero value that is more suitable for machine learning models\.
+ NumericPassthrough — Passes input as is into the model\.
+ Imputer — Fills in missing values and NaN values\.
+ Normalizer — Normalizes values that often improves the performance of many machine learning algorithms\.

Amazon Redshift ML stores the trained transformers, and automatically applies them as part of the prediction query\. You don't need to specify them when generating predictions from the model\. 

## CREATE XGBoost models with AUTO OFF<a name="r_auto_off_create_model"></a>

The AUTO OFF CREATE MODEL has generally different objectives from the default CREATE MODEL\.

As an advanced user who already know the model type that you want and hyperparameters to use when training these models, you can use CREATE MODEL with AUTO OFF to turn off the CREATE MODEL automatic discovery of preprocessors and hyperparameters\. To do so, you explicitly specify the model type\. XGBoost is currently the only model type supported when AUTO is set to OFF\. You can specify hyperparameters\. Amazon Redshift uses default values for any hyperparameters that you specified\.  

### CREATE MODEL with AUTO OFF syntax<a name="r_auto_off-create-model-synposis"></a>

```
CREATE MODEL model_name
FROM { table_name | (select_statement ) }
TARGET column_name    
FUNCTION function_name
IAM_ROLE 'iam_role_arn'
AUTO OFF
MODEL_TYPE XGBOOST
OBJECTIVE { 'reg:squarederror' | 'reg:squaredlogerror' | 'reg:logistic' | 
            'reg:pseudohubererror' | 'reg:tweedie' | 'binary:logistic' | 'binary:hinge' | 
            'multi:softmax' | 'rank:pairwise' | 'rank:ndcg' }
HYPERPARAMETERS DEFAULT EXCEPT (
    NUM_ROUND '10',
    ETA '0.2',
    NUM_CLASS '10',
    (, ...)
)
PREPROCESSORS 'none'
SETTINGS (
  S3_BUCKET 'bucket', |
  S3_GARBAGE_COLLECT { ON | OFF }, |
  KMS_KEY_ID 'kms_key_id', |
  MAX_CELLS integer, |
  MAX_RUNTIME integer (, ...)
)
```

### CREATE XGBoost models with AUTO OFF parameters<a name="r_auto_off-create-model-parameters"></a>

 *AUTO OFF*   
Turns off CREATE MODEL automatic discovery of preprocessor, algorithm and hyper\-parameters selection\.

MODEL\_TYPE XGBOOST  
Specifies to use XGBOOST to train the model\. 

OBJECTIVE str  
Specifies an objective recognized by the algorithm\. Amazon Redshift supports reg:squarederror, reg:squaredlogerror, reg:logistic, reg:pseudohubererror, reg:tweedie, binary:logistic, binary:hinge, multi:softmax\. For more information about these objectives, see [Learning task parameters](https://xgboost.readthedocs.io/en/latest/parameter.html#learning-task-parameters) in the XGBoost documentation\.

HYPERPARAMETERS \{ DEFAULT \| DEFAULT EXCEPT \( key ‘value’ \(,\.\.\) \) \}  
Specifies whether the default XGBoost parameters are used or over\-ridden by user\-specified values\. The values must be enclosed with single quotes\. Following are examples of parameters for XGBoost and their defaults\.      
[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/redshift/latest/dg/r_CREATE_MODEL.html)

The following example prepares data for XGBoost\.

```
DROP TABLE IF EXISTS abalone_xgb;

CREATE TABLE abalone_xgb (
length_val float, 
diameter float, 
height float,
whole_weight float, 
shucked_weight float, 
viscera_weight float,
shell_weight float, 
rings int,
record_number int);

COPY abalone_xgb
FROM 's3://redshift-downloads/redshift-ml/abalone_xg/'
REGION 'us-east-1'
IAM_ROLE 'arn:aws:iam::467896856988:role/Redshift-ML'
IGNOREHEADER 1 CSV;
```

The following example creates an XGBoost model with specified advanced options, such as MODEL\_TYPE, OBJECTIVE, and PREPROCESSORS\.

```
DROP MODEL abalone_xgboost_multi_predict_age;

CREATE MODEL abalone_xgboost_multi_predict_age
FROM ( SELECT length_val,
              diameter,
              height,
              whole_weight,
              shucked_weight,
              viscera_weight,
              shell_weight,
              rings 
        FROM abalone_xgb WHERE record_number < 2500 )
TARGET rings FUNCTION ml_fn_abalone_xgboost_multi_predict_age
IAM_ROLE 'arn:aws:iam::XXXXXXXXXXXX:role/Redshift-ML'
AUTO OFF
MODEL_TYPE XGBOOST
OBJECTIVE 'multi:softmax'
PREPROCESSORS 'none'
HYPERPARAMETERS DEFAULT EXCEPT (NUM_ROUND '100', NUM_CLASS '30')
settings (S3_BUCKET 'your-bucket');
```

The following example uses an inference query to predict the age of the fish with a record number greater than 200\. It uses the function ml\_fn\_abalone\_xgboost\_multi\_predict\_age created from the above command\. 

```
select ml_fn_abalone_xgboost_multi_predict_age(length_val, 
                                               diameter, 
                                               height, 
                                               whole_weight, 
                                               shucked_weight, 
                                               viscera_weight, 
                                               shell_weight)+1.5 as age 
from abalone_xgb where record_number > 2500;
```

## Bring your own model \(BYOM\)<a name="r_byom_create_model"></a>

Amazon Redshift ML supports using BYOM for local inference\. The following summarizes the options of the CREATE MODEL syntax for BYOM\.

### CREATE MODEL syntax for local inference<a name="r_local-create-model"></a>

The following describes the CREATE MODEL syntax for local inference\.

```
CREATE MODEL model_name
FROM 'job_name'
FUNCTION function_name ( data_type [, ...] )
RETURNS data_type
IAM_ROLE 'iam-role-arn'
[ SETTINGS (
  S3_BUCKET 'bucket', |
  KMS_KEY_ID 'kms_string')
];
```

Amazon Redshift currently only supports pretrained XGBoost models for BYOM\. You can import SageMaker Autopilot and direct Amazon SageMaker trained models for local inference using this path\. 

#### CREATE MODEL parameters for local inference<a name="r_local-create-model-parameters"></a>

 *model\_name*   
The name of the model\. The model name in a schema must be unique\.

FROM *'job\_name'*   
The FROM clause uses an Amazon SageMaker job name as the input\. The job name can either be a training job name or an AutoML job name\.

FUNCTION *function\_name* \( *data\_type* \[, \.\.\.\] \)  
The name of the function to be created and the data types of the input arguments\. You can provide a schema name\.

RETURNS *data\_type*  
The data type of the value returned by the function\.

IAM\_ROLE *'iam\_role\_arn'*  
The Amazon Resource Name \(ARN\) for an AWS Identity and Access Management \(IAM\) role that your cluster uses for authentication and authorization\.   
The following shows the syntax for the IAM\_ROLE parameter string for a single ARN\.  

```
IAM_ROLE 'arn:aws:iam::aws-account-id:role/role-name'
```

SETTINGS \( S3\_BUCKET *'bucket'*, \| KMS\_KEY\_ID *'kms\_string'*\)  
\(Optional\) The S3\_BUCKET clause specifies the Amazon S3 location that is used to store intermediate results\. If you don't choose an Amazon S3 bucket, Amazon Redshift uses the location where the trained model is located to store intermediate results\.  
\(Optional\) The KMS\_KEY\_ID clause specifies if Amazon Redshift uses server\-side encryption with an AWS KMS key to protect data at rest\. Data in transit is protected with Secure Sockets Layer \(SSL\)\.  
For more information, see [CREATE MODEL with user guidance](#r_user_guidance_create_model)\.

#### CREATE MODEL for local inference example<a name="r_local-create-model-example"></a>

The following example creates a model that has been previously trained in Amazon SageMaker, outside of Amazon Redshift\. Because the model type is supported by Amazon Redshift ML for local inference, the following CREATE MODEL creates a function that can be used locally in Amazon Redshift\. You can provide a SageMaker training job name\.

```
CREATE MODEL customer_churn
FROM 'training-job-customer-churn-v4'
FUNCTION customer_churn_predict (varchar, int, float, float)
RETURNS int
IAM_ROLE 'arn:aws:iam::123456789012:role/Redshift-ML';
```

After the model is created, you can use the function *customer\_churn\_predict* with the specified argument types to make predictions\.

## Full CREATE MODEL<a name="r_full_create_model"></a>

The following summarizes the basic options of the full CREATE MODEL syntax\.

### Full CREATE MODEL syntax<a name="r_auto_off-create-model-synposis"></a>

The following is the full syntax of the CREATE MODEL statement\. This syntax is used when the AUTO ON semiautomatic [CREATE MODEL with user guidance](#r_user_guidance_create_model) and the AUTO OFF [CREATE XGBoost models with AUTO OFF](#r_auto_off_create_model) work together\. This syntax also includes the CREATE MODEL statement for BYOM\.

```
CREATE MODEL model_name
FROM { table_name | ( select_statement )  | 'job_name' }
[ TARGET column_name ]   
FUNCTION function_name ( [data_type] [, ...] ) 
IAM_ROLE 'iam_role_arn' 
[ AUTO ON / OFF ]
  -- default is AUTO ON 
[ MODEL_TYPE { XGBOOST } ]
  -- not required for non AUTO OFF case, default is the list of all supported type
  -- required for AUTO OFF 
[ PROBLEM_TYPE ( REGRESSION | BINARY_CLASSIFICATION | MULTICLASS_CLASSIFICATION ) ]
  -- not supported when AUTO OFF 
[ OBJECTIVE ( 'MSE' | 'Accuracy' | 'F1' | 'F1_Macro' | 'AUC' |
              'reg:squarederror' | 'reg:squaredlogerror'| 'reg:logistic'|
              'reg:pseudohubererror' | 'reg:tweedie' | 'binary:logistic' | 'binary:hinge',
              'multi:softmax' ) ]
  -- for AUTO ON: first 5 are valid
  -- for AUTO OFF: 6-13 are valid
[ PREPROCESSORS 'string' ]
  -- required for AUTO OFF, when it has to be 'none'
  -- optional for AUTO ON
[ HYPERPARAMETERS { DEFAULT | DEFAULT EXCEPT ( Key 'value' (,...) ) } ]
  -- support XGBoost hyperparameters, except OBJECTIVE
  -- required and only allowed for AUTO OFF
  -- default NUM_ROUND is 100
  -- NUM_CLASS is required if objective is multi:softmax (only possible for AUTO OFF)
 [ SETTINGS (
   S3_BUCKET 'bucket',  |
    -- required 
  KMS_KEY_ID 'kms_string', |
    -- optional
  S3_GARBAGE_COLLECT on / off, |
    -- optional, defualt is on.
  MAX_CELLS integer, |
    -- optional, default is 1,000,000
  MAX_RUNTIME integer (, ...)
    -- optional, default is 5400 (1.5 hours) ]
)
```

## Usage notes<a name="r_create_model_usage_notes"></a>

When using CREATE MODEL, consider the following:
+ The CREATE MODEL statement operates in an asynchronous mode and returns upon the export of training data to Amazon S3\. The remaining steps of training in Amazon SageMaker occur in the background\. While training is in progress, the corresponding inference function is visible but can't be run\. You can query [STV\_ML\_MODEL\_INFO](r_STV_ML_MODEL_INFO.md) to see the state of training\. The training can run for up to 90 minutes in the background, by default in the Auto model and can be extended\. To cancel the training, simply run the [DROP MODEL](r_DROP_MODEL.md) command\.
+ CREATE MODEL operates in a synchronous mode and can run for up to 90 minutes by default in the AUTO mode\.
+ The Amazon Redshift cluster that you use to create the model and the Amazon S3 bucket that is used to stage the training data and model artifacts must be in the same AWS Region\.
+ During the model training, Amazon Redshift and SageMaker store intermediate artifacts in the Amazon S3 bucket that you provide\. By default, Amazon Redshift performs garbage collection at the end of CREATE MODEL and removes those objects from Amazon S3\. To retain those artifacts on Amazon S3, you can set the S3\_GARBAGE COLLECT OFF option\.
+ You must use at least 500 rows in the training data provided in the FROM clause\.
+ You can only specify up to 32 feature \(input\) columns in the FROM \{ table\_name \| \( select\_query \) \} clause when using the CREATE MODEL statement\.
+ For AUTO ON, the column types that you can use as the training set are SMALLINT, INTEGER, BIGINT, DECIMAL, REAL, DOUBLE, BOOLEAN, CHAR, and VARCHAR\. For AUTO OFF, the column types that you can use as the training set are SMALLINT, INTEGER, BIGINT, DECIMAL, REAL, DOUBLE, and BOOLEAN\.
+ You can't use DECIMAL as the target column type\.
+ The CREATE MODEL statement execution returns as soon as the training data have been computed and exported on the Amazon S3 bucket\. After that point, you can check the status of the training using the SHOW MODEL command\. For more information about SHOW MODEL, see [SHOW MODEL](r_SHOW_MODEL.md)\.
+ Local BYOM supports the same kind of models that Amazon Redshift ML supports for non\-BYOM cases\. Amazon Redshift supports plain XGBoost models \(using XGBoost version 1\.0 or later\) without preprocessors and XGBoost models trained by Amazon SageMaker Autopilot\. It supports the latter with preprocessors that Autopilot has specified that are also supported by Amazon SageMaker Neo\.