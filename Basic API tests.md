##models name
curl http://169.254.36.7:8000/v1/models

##simple completions test
`curl http://169.254.36.7:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/models/Nemotron120B",
    "prompt": "The capital of France is",
    "max_tokens": 20
  }'`


##Then try a chat request:
`curl http://169.254.36.7:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "/models/Nemotron120B",
    "messages": [
      {"role": "user", "content": "Write one short sentence confirming you are running."}
    ],
    "max_tokens": 50,
    "temperature": 0.2
  }'`
 