
## Example 1 - Don't use `setState` inside another `setState`

```typescript
setFormData((prevFormData) => {
  const updatedFormData = prevFormData.map((item) => {
    if (item.id === id) {
      return { ...item, value };
    }
    return item;
  });

  // Validate the field
  const errorMessage = validateField(
    id,
    value,
    prevFormData.find((item) => item.id === id)?.required || false,
  );

  setErrors((prevErrors) => {
    const newErrors = { ...prevErrors };
    if (errorMessage) {
      newErrors[id] = errorMessage;
    } else {
      delete newErrors[id];
    }
    return newErrors;
  });

  return updatedFormData;
});
```

```typescript
let newFormData = [...formData];
const updatedFormData = newFormData.map((item) => {
  if (item.id === id) {
    return { ...item, value }
  }
  return item
})
const errorMessage = validateField(
  id,
  value,
  newFormData.find((item) => item.id === id)?.required || false,
)
const newErrors = { ...errors }
if (errorMessage) {
  newErrors[id] = errorMessage
} else {
  delete newErrors[id]
}
setErrors(newErrors);
setFormData(updatedFormData);
```
