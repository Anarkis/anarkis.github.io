---
layout: post
title: 'Reverse engineering Zinio app auth'
tags: [ 'zinio', 'reverse', 'python', 'auth']
featured_image_thumbnail: assets/images/posts/zinio_thumbnail.jpg
featured_image: assets/images/posts/zinio.jpg
featured: true
hidden: true
---
I subscribed some time ago to a climbing magazine through Zinio. And I wanted to see how they implemented the authentication system in their app. (Lately in Spain we had some huge problem with the same topic: [Metrovalencia](https://valenciaplaza.com/la-app-de-metrovalencia-deja-al-descubierto-los-datos-de-60000-usuarios))

Zinio app is based in [Node.js](https://nodejs.org/en/). Wasn't difficult to see under the hood.
After so time I figure out the headers request
```python
headers = {
  'authority':'www.zinio.com',
  'method':'POST',
  'path':'/api/login?project=99',
  'scheme':'https',
  "authorization": "",
  "X-ZINIO-User-Id": "",
  "User-Agent": "Mozilla/5.0 ...",
  "accept" : "*/*",
  "accept-encoding" : "gzip, deflate, br",
  "accept-language" : "en-US,en;q=0.9",
  "content-length" : "53",
  "content-type" : "application/json",
  "origin" : "null"
}
```
<!--more-->
I found quite interesting that they have in the code **fixed credentials** for identify yourself as a valid client.
*Could those fixed credentials be used in other places?*

```python
  "client_id": 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX',
  "client_secret": 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'
```

Next step was discover how they prepare the authorization field in the headers. They do something like this:

```python
  cred_concat = fixed_credentials['client_id'] + fixed_credentials['client_secret']
  cred_hash = hashlib.sha1(cred_concat.encode('utf8'))
  cred_hash = cred_hash.hexdigest()

  headers['authorization'] = 'desktop ' + cred_hash
```

The only thing that is missing is the user and pwd.
```python
  body_cred = {
      'username' : 'secret@email.com',
      'password' : 'supersecretpwd'
  }
```

We are ready to do our successful request
```python
  response = requests.post(url_login, headers=headers, params={"project":"99"}, data=json.dumps(body_cred))
  response_json = response.json()
```

And we get a nice response, I point out the most significative fields
```python
  {
    'data':
      {
        'user': {
           'email': 'secret@email.com',
           'user_id_string': 'XXXXXXXXXXXXXXXXXXX'
        },
        'token': {
          'token_type': 'bearer',
          'access_token': 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX',
          'expires_in': 7200,
          'EXPIRES_AT': '2019-03-30T10:36:37.699Z'
        },
        'refreshToken': 'secret-string-of-243-characters'},
        'status': True
  }
```
Now, we know:
  * Zinio uses a bearer auth system
  * We know our user_id after a successful login

After checking how is the **refreshToken** with [JWT](https://jwt.io/) we see the below structure

###header
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
###payload
```json
{
  "payload": {
    "userId": "XXXXXXXXXXXXXXXXXXX",
    "token": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  },
  "iat": 1213935449,
  "exp": 1215231449
}
```

From now on, we could create a nice code which download all our magazines :)
