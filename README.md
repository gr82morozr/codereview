"""



def highlight_case_insensitive(text, keyword):
    # Use re.escape to handle special characters in keyword
    pattern = re.compile(re.escape(keyword), re.IGNORECASE)

    # Use a lambda to wrap the *matched text* (preserves original case)
    return pattern.sub(lambda m: f"<mark>{m.group(0)}</mark>", text)

# Example usage:
text = "John went to the market. john is a common name."
keyword = "john"

result = highlight_case_insensitive(text, keyword)
print(result)


"""
