- [Overview](#overview)
- [Installation Instructions](#installation-instructions)
- [How to configure database and schema for the masking policy ?](#how-to-configure-database-and-schema-for-the-masking-policy-)
- [How to apply masking policy ?](#how-to-apply-masking-policy-)
- [How to remove masking policy ?](#how-to-remove-masking-policy-)
- [How to validate masking policy ?](#how-to-validate-masking-policy-)
- [Process flow](#process-flow)
  - [Create masking policy](#create-masking-policy)
  - [Apply masking policy](#apply-masking-policy)
- [Known Errors and Solutions](#known-errors-and-solutions)
- [Credits](#credits)
- [References](#references)
- [Contributions](#contributions)

# Overview
This dbt package contains macros that can be (re)used across dbt projects with snowflake. `dbt_snow_mask` will help to apply [Dynamic Data Masking](https://docs.snowflake.com/en/user-guide/security-column-ddm-use.html) using [dbt meta](https://docs.getdbt.com/reference/resource-properties/meta).

# Installation Instructions

- Add the package into your project.

  **Example** : packages.yml

  ```bash
     - git: "https://github.com/entechlog/dbt-snow-mask.git"
       revision: 0.1.7
  ```

  ```bash
     - package: entechlog/dbt_snow_mask
       version: 0.1.7
  ```

> ✅ Packages can be added to your project using either of above options  
> ✅ Please refer to the release version of this repo/dbt hub for the latest revision

- This package uses [dbt_utils](https://hub.getdbt.com/dbt-labs/dbt_utils/latest/) package. When using `dbt_snow_mask` in your project, please install [dbt_utils](https://hub.getdbt.com/dbt-labs/dbt_utils/latest/) as well. You will get an error if you attempt to use this package without installing `dbt_snow_mask`

  ```bash
     - package: dbt-labs/dbt_utils
       version: 0.8.2
  ```

> ✅ Please refer to the release version in dbt hub for the latest revision


# How to configure database and schema for the masking policy ?

By default this process creates the masking policies in same directory as the database objects. You can change this default behavior by using following parameters in your `dbt_project.yml` 

* `use_common_masking_policy_db` (optional): Flag to enable the usage of a common db/schema for all masking policies. Valid values are “True” OR "False"
* `common_masking_policy_db` (optional): The database name for creating masking policies
* `common_masking_policy_schema` (optional): The schema name for creating masking policies

**Example** : var block in dbt_project.yml

```yaml
vars:
  use_common_masking_policy_db: "True"
  common_masking_policy_db: "DEMO_DB"
  common_masking_policy_schema: "COMPLIANCE"
```

# How to apply masking policy ?

- Masking is controlled by [meta](https://docs.getdbt.com/reference/resource-properties/meta) in [dbt resource properties](https://docs.getdbt.com/reference/declaring-properties) for sources and models. 

- Decide you masking policy name and add the key `masking_policy` in the column which has to be masked.
  
  **Example** : source.yml

  ```yaml
  sources:
    - name: raw_sakila
      tables:
        - name: customer
          columns:
            - name: first_name
              meta:
                  masking_policy: mp_encrypt_pii
  ```
  
  **Example** : model.yml

  ```yaml
  models:
    - name: stg_customer
      columns:
        - name: email
          meta:
            masking_policy: mp_encrypt_pii
  ```

- Create a new `.sql` file with the name `create_masking_policy_<masking-policy-name-from-meta>.sql` and the sql for masking policy definition. Its important for macro to follow this naming standard.
  
  **Example** : create_masking_policy_mp_encrypt_pii.sql

  ```sql
  {% macro create_masking_policy_mp_encrypt_pii(node_database,node_schema) %}

  CREATE MASKING POLICY IF NOT EXISTS {{node_database}}.{{node_schema}}.mp_encrypt_pii AS (val string) 
    RETURNS string ->
        CASE WHEN CURRENT_ROLE() IN ('ANALYST') THEN val 
             WHEN CURRENT_ROLE() IN ('DEVELOPER') THEN SHA2(val)
        ELSE '**********'
        END

  {% endmacro %}
  ```

> Its good to keep the masking policy ddl organized in a directory say `\macros\snow-mask-ddl`

- Create the masking policies by running below command  
  
  | Resource Type | Command                                                                         |
  | ------------- | ------------------------------------------------------------------------------- |
  | sources       | `dbt run-operation create_masking_policy --args '{"resource_type": "sources"}'` |
  | models        | `dbt run-operation create_masking_policy --args '{"resource_type": "models"}'`  |

- Alternatively, you can also create the masking policies by specifying `pre-hook` OR `on-run-start` in your `dbt_project.yml`
  
  ```yaml
  on-run-start:
    - "{{ dbt_snow_mask.create_masking_policy('models')}}"
    - "{{ dbt_snow_mask.create_masking_policy('sources')}}"
  ```

- Apply the masking policy by running below commands  

  | Resource Type | Command                                                                        |
  | ------------- | ------------------------------------------------------------------------------ |
  | sources       | `dbt run-operation apply_masking_policy --args '{"resource_type": "sources"}'` |
  | models        | `dbt run -- model <model-name>`                                                |

- Alternatively, you can also apply the masking policies by specifying below `post-hook` OR `on-run-end` to `dbt_project.yml`
  
  **Example** : dbt_project.yml

  ```yaml
  models:
    post-hook: 
      - "{{ dbt_snow_mask.apply_masking_policy('models') }}"
  ```

# How to remove masking policy ?

- Remove the masking policy applied by this package by running below commands  

  | Resource Type | Command                                                                          |
  | ------------- | -------------------------------------------------------------------------------- |
  | sources       | `dbt run-operation unapply_masking_policy --args '{"resource_type": "sources"}'` |
  | models        | `dbt run-operation unapply_masking_policy --args '{"resource_type": "models"}'`  |

- Alternatively, you can also apply the unmasking policies by specifying below `post-hook` OR `on-run-end` to `dbt_project.yml`
  
  **Example** : dbt_project.yml

  ```yaml
  models:
    post-hook: 
      - "{{ dbt_snow_mask.unapply_masking_policy('models') }}"
  ```

# How to validate masking policy ?

```sql
-- Show masking policy
SHOW MASKING POLICIES;

-- Describe masking policy
DESCRIBE MASKING POLICY <masking-policy-name>;

-- Show masking policy references
USE DATABASE <database-name>;

USE SCHEMA INFORMATION_SCHEMA;

SELECT *
  FROM TABLE(INFORMATION_SCHEMA.POLICY_REFERENCES(POLICY_NAME => '<database-name>.<schema-name>.<masking-policy-name>'));
```

# Process flow
## Create masking policy

```mermaid
graph TD
    A[create_masking_policy] --> |resource_type='sources',meta_key='masking_policy'| B[get_masking_policy_list_for_sources]
    A[create_masking_policy] --> |resource_type='models',meta_key='masking_policy'| C[get_masking_policy_list_for_models]
    B --> |database, schema| D[create_schema]
    C --> |database, schema| D[create_schema]
    D --> |policy_name| E[call_masking_policy_macro]
```

## Apply masking policy

```mermaid
graph TD
    A[apply_masking_policy] --> |resource_type='sources',meta_key='masking_policy'| B[apply_masking_policy_list_for_sources]
    A[apply_masking_policy] --> |resource_type='models',meta_key='masking_policy'| C[apply_masking_policy_list_for_models]
    B --> |meta_key| D[confirm masking policy is avaliable in db]
    C --> |meta_key| D[confirm masking policy is avaliable in db]
    D --> E[alter statement to set masking policy]
```

# Known Errors and Solutions
| Error                                                               | Solution                                                                 |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| dict object' has no attribute 'create_masking_policy_mp_encrypt_pi' | Typo in yaml for masking_policy, mp_encrypt_pi instead of mp_encrypt_pii |

# Credits
This package was created using examples from [Serge](https://www.linkedin.com/in/serge-gekker-912b9928/) and [Matt](https://www.linkedin.com/in/matt-winkler-4024263a/). Please see the [contributors](https://github.com/entechlog/dbt-snow-mask/graphs/contributors) for full list of users who have contributed to this project.

# References
- https://docs.snowflake.com/en/user-guide/security-column-ddm-intro.html
- https://getdbt.slack.com/archives/CJN7XRF1B/p1609177817234800

# Contributions
Contributions to this package are welcomed. Please create issues for bugs or feature requests for enhancement ideas or PRs for any enhancement contributions.