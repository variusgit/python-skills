## Rules

1. Do not add new dependencies to the project.

2. Do not add project-wide Pyright ignore rules or disable type-checking errors globally.

3. Do not add inline lint/type-check ignores without human approval.  
   If an ignore is approved, it must target only the specific error being suppressed.  
   Example template:
   `# pyright: ignore[reportReturnType]  # type checker doesn't detect env loading`

4. Do not create new loggers. Always use the root logger.  
   Example:
   `logging.info("...")`

5. Log messages must start with a lowercase letter, except for abbreviations.

6. Do not use deprecated typing classes. Use modern Python type hints.  
   Examples:
   - `list` instead of `typing.List`
   - `dict` instead of `typing.Dict`
   - `A | B` instead of `typing.Union[A, B]`
   - `A | None` instead of `typing.Optional[A]`

7. In tests, always log relevant values before asserting them.  
   For example, before asserting a response status code, log the full response.

8. Do not use raw integers for HTTP status codes.  
   Always import them from FastAPI:
   `from fastapi import status`

9. Always declare all possible responses (including error cases) in the route decorator.  
   Use the `responses` parameter in `@router.<verb>`.  
   Example:

   ```python
   @router.get(
       "...",
       responses={
           status.HTTP_404_NOT_FOUND: {
               "description": "Communication not found or campaign not found"
           }
       },
   )