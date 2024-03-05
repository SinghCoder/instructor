# Response Model

Defining LLM output schemas in Pydantic is done via `pydantic.BaseModel`. To learn more about models in Pydantic, check out their [documentation](https://docs.pydantic.dev/latest/concepts/models/).

After defining a Pydantic model, we can use it as the `response_model` in your client `create` calls to OpenAI or any other supported model. The job of the `response_model` parameter is to:

- Define the schema and prompts for the language model
- Validate the response from the API
- Return a Pydantic model instance.

## Prompting

When defining a response model, we can use docstrings and field annotations to define the prompt that will be used to generate the response.

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    """
    This is the prompt that will be used to generate the response.
    Any instructions here will be passed to the language model.
    """

    name: str = Field(description="The name of the user.")
    age: int = Field(description="The age of the user.")
```

Here all docstrings, types, and field annotations will be used to generate the prompt. The prompt will be generated by the `create` method of the client and will be used to generate the response.

## Optional Values

If we use `Optional` and `default`, they will be considered not required when sent to the language model

```python
from pydantic import BaseModel, Field
from typing import Optional


class User(BaseModel):
    name: str = Field(description="The name of the user.")
    age: int = Field(description="The age of the user.")
    email: Optional[str] = Field(description="The email of the user.", default=None)
```

## Dynamic model creation

There are some occasions where it is desirable to create a model using runtime information to specify the fields. For this, Pydantic provides the create_model function to allow models to be created on the fly:

```python
from pydantic import BaseModel, create_model


class FooModel(BaseModel):
    foo: str
    bar: int = 123


BarModel = create_model(
    'BarModel',
    apple=(str, 'russet'),
    banana=(str, 'yellow'),
    __base__=FooModel,
)
print(BarModel)
#> <class '__main__.BarModel'>
print(BarModel.model_fields.keys())
#> dict_keys(['foo', 'bar', 'apple', 'banana'])
```

??? notes "When would I use this?"

    Consider a situation where the model is dynamically defined, based on some configuration or database. For example, we could have a database table that stores the properties of a model for
    some model name or id. We could then query the database for the properties of the model and use that to create the model.

    ```sql
    SELECT property_name, property_type, description
    FROM prompt
    WHERE model_name = {model_name}
    ```

    We can then use this information to create the model.

    ```python
    from pydantic import BaseModel, create_model
    from typing import List

    types = {
        'string': str,
        'integer': int,
        'boolean': bool,
        'number': float,
        'List[str]': List[str],
    }

    # Mocked cursor.fetchall()
    cursor = [
        ('name', 'string', 'The name of the user.'),
        ('age', 'integer', 'The age of the user.'),
        ('email', 'string', 'The email of the user.'),
    ]

    BarModel = create_model(
        'User',
        **{
            property_name: (types[property_type], description)
            for property_name, property_type, description in cursor
        },
        __base__=BaseModel,
    )

    print(BarModel.model_json_schema())
    """
    {
        'properties': {
            'name': {'default': 'The name of the user.', 'title': 'Name', 'type': 'string'},
            'age': {'default': 'The age of the user.', 'title': 'Age', 'type': 'integer'},
            'email': {
                'default': 'The email of the user.',
                'title': 'Email',
                'type': 'string',
            },
        },
        'title': 'User',
        'type': 'object',
    }
    """
    ```

    This would be useful when different users have different descriptions for the same model. We can use the same model but have different prompts for each user.

## Adding Behavior

We can add methods to our Pydantic models, just as any plain Python class. We might want to do this to add some custom logic to our models.

```python
from pydantic import BaseModel
from typing import Literal

from openai import OpenAI

import instructor

client = instructor.patch(OpenAI())


class SearchQuery(BaseModel):
    query: str
    query_type: Literal["web", "image", "video"]

    def execute(self):
        print(f"Searching for {self.query} of type {self.query_type}")
        #> Searching for cat pictures of type image
        return "Results for cat"


query = client.chat.completions.create(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Search for a picture of a cat"}],
    response_model=SearchQuery,
)

results = query.execute()
print(results)
#> Results for cat
```

Now we can call `execute` on our model instance after extracting it from a language model. If you want to see more examples of this checkout our post on [RAG is more than embeddings](../blog/posts/rag-and-beyond.md)