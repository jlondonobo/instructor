# Validation and Reasking

Instead of framing "self-critique" or "self-reflection" in AI as new concepts, we can view them as validation errors with clear error messages that the systen can use to self correct.

## Pydantic

Pydantic offers an customizable and expressive validation framework for Python. Instructor leverages Pydantic's validation framework to provide a uniform developer experience for both code-based and LLM-based validation, as well as a reasking mechanism for correcting LLM outputs based on validation errors. To learn more check out the [Pydantic docs](https://docs.pydantic.dev/latest/concepts/validators/) on validators.

!!! note "Good llm validation is just good validation"

    If you want to see some more examples on validators checkout our blog post [Good llm validation is just good validation](https://jxnl.github.io/instructor/blog/2023/10/23/good-llm-validation-is-just-good-validation/)

### Code-Based Validation Example

First define a Pydantic model with a validator using the `Annotation` class from `typing_extensions`.

Enforce a naming rule using Pydantic's built-in validation:

```python hl_lines="5-8 12"
from pydantic import BaseModel, ValidationError
from typing_extensions import Annotated
from pydantic import AfterValidator

def name_must_contain_space(v: str) -> str:
    if " " not in v:
        raise ValueError("Name must contain a space.")
    return v.lower()

class UserDetail(BaseModel):
    age: int
    name: Annotated[str, AfterValidator(name_must_contain_space)]

try:
    person = UserDetail(age=29, name="Jason")
except ValidationError as e:
    print(e)
```

#### Output for Code-Based Validation

```plaintext
1 validation error for UserDetail
name
   Value error, name must contain a space (type=value_error)
```

As we can see, Pydantic raises a validation error when the name attribute does not contain a space. This is a simple example, but it demonstrates how Pydantic can be used to validate attributes of a model.

### LLM-Based Validation Example

LLM-based validation can also be plugged into the same Pydantic model. Here, if the answer attribute contains content that violates the rule "don't say objectionable things," Pydantic will raise a validation error.

```python hl_lines="9 15"
import instructor

from openai import OpenAI
from instructor import llm_validator
from pydantic import BaseModel, ValidationError, BeforeValidator
from typing_extensions import Annotated

# Apply the patch to the OpenAI client
client = instructor.patch(OpenAI())


class QuestionAnswer(BaseModel):
    question: str
    answer: Annotated[
        str,
        BeforeValidator(llm_validator("don't say objectionable things", openai_client=client))
    ]

try:
    qa = QuestionAnswer(
        question="What is the meaning of life?",
        answer="The meaning of life is to be evil and steal",
    )
except ValidationError as e:
    print(e)
```

#### Output for LLM-Based Validation

Its important to not here that the error message is generated by the LLM, not the code, so it'll be helpful for re asking the model.

```plaintext
1 validation error for QuestionAnswer
answer
   Assertion failed, The statement is objectionable. (type=assertion_error)
```

## Using Reasking Logic to Correct Outputs

Validators are a great tool for ensuring some property of the outputs. When you use the `patch()` method with the `openai` client, you can use the `max_retries` parameter to set the number of times you can reask the model to correct the output.

Its a great layer of defense against bad outputs of two forms.

1. Pydantic Validation Errors (code or llm based)
2. JSON Decoding Errors (when the model returns a bad response)

### Step 1: Define the Response Model with Validators

Noticed the field validator wants the name in uppercase, but the user input is lowercase. The validator will raise a `ValueError` if the name is not in uppercase.

```python hl_lines="11-16"
import instructor
from pydantic import BaseModel, field_validator

# Apply the patch to the OpenAI client
client = instructor.patch(OpenAI())

class UserDetails(BaseModel):
    name: str
    age: int

    @field_validator("name")
    @classmethod
    def validate_name(cls, v):
        if v.upper() != v:
            raise ValueError("Name must be in uppercase.")
        return v
```

### Step 2. Using the Client with Retries

Here, the `UserDetails` model is passed as the `response_model`, and `max_retries` is set to 2.

```python hl_lines="4 10"
model = client.chat.completions.create(
    model="gpt-3.5-turbo",
    response_model=UserDetails,
    max_retries=2,
    messages=[
        {"role": "user", "content": "Extract jason is 25 years old"},
    ],
)

assert model.name == "JASON"
```

### What happens behind the scenes?

Behind the scenes, the `instructor.patch()` method adds a `max_retries` parameter to the `openai.ChatCompletion.create()` method. The `max_retries` parameter will trigger up to 2 reattempts if the `name` attribute fails the uppercase validation in `UserDetails`.

```python
try:
    ...
except (ValidationError, JSONDecodeError) as e:
    kwargs["messages"].append(response.choices[0].message)
    kwargs["messages"].append(
        {
            "role": "user",
            "content": f"Please correct the function call; errors encountered:\n{e}",
        }
    )
```

## Advanced Validation Techniques

The docs are currently incomplete, but we have a few advanced validation techniques that we're working on documenting better, for a example of model level validation, and using a validation context check out our example on [verifying citations](../examples/exact_citations.md) which covers

1. Validate the entire object with all attributes rather than one attribute at a time
2. Using some 'context' to validate the object, in this case we use the `context` to check if the citation existed in the original text.

## Takeaways

By integrating these advanced validation techniques, we not only improve the quality and reliability of LLM-generated content but also pave the way for more autonomous and effective systems.