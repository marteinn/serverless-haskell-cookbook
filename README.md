# The missing cookbook for serverless-haskell

When I started toying with [serverless-haskell](https://hackage.haskell.org/package/serverless-haskell-0.8.7/) I noticed that examples were hard to come by, so this is a collection of examples I would have wanted when I got started. Enjoy!

## Recipes
- [Retrieve a raw text response and return it](#retrieve-a-raw-text-response-and-return-it)
- [Accept a JSON payload, extract a param and return it](#accept-a-json-payload-extract-a-param-and-return-it)
- [Return a list of strings as json](#return-a-list-of-strings-as-json)
- [Return a data type as json](#return-a-data-type-as-json)
- [Retrieve query var and display it](#retrieve-query-var-and-display-it)
- [Get header from request and display in response](#get-header-from-request-and-display-in-response)
- [Returning status code 401 or 200 if correct header token was passed](#returning-status-code-401-or-200-if-correct-header-token-was-passed)
- [Adding a custom header to a response](#adding-a-custom-header-to-a-response)
- [Retrieve path parameter and show different messages based on its value](#retrieve-path-parameter-and-show-different-messages-based-on-its-value)
- [Handling x-www-form-urlencoded data](#handling-x-www-form-urlencoded-data)

#### Retrieve a raw text response and return it

```haskell
{-# LANGUAGE OverloadedStrings #-}
module Main where

import AWSLambda.Events.APIGateway
import Control.Lens
import Data.Semigroup
import Data.Text                   (Text, pack)
import Data.Aeson.TextValue

main :: IO ()
main = apiGatewayMain returnRawBody

returnRawBody :: APIGatewayProxyRequest Text -> IO (APIGatewayProxyResponse Text)
returnRawBody request = do
  let
    rawBody =
        case (request ^. agprqBody) of
            Just (TextValue value) -> value
            Nothing -> ""
  do
    pure $ responseOK 
        & responseBody ?~ rawBody
```

#### Accept a JSON payload, extract a param and return it

```haskell
{-# LANGUAGE OverloadedStrings, DeriveGeneric #-}
module Main where

import AWSLambda.Events.APIGateway
import Control.Lens
import Data.Semigroup
import Data.Aeson
import Data.Aeson.Embedded
import GHC.Generics

main :: IO ()
main = apiGatewayMain acceptJsonData

data Person = Person { firstName :: String } deriving (Show, Generic)

instance FromJSON Person
instance ToJSON Person

acceptJsonData :: APIGatewayProxyRequest (Embedded Person) -> IO (APIGatewayProxyResponse String)
acceptJsonData request = do
  let
    person =
      case (request ^. requestBody) of
        Just (Embedded value) -> value
        Nothing -> Person { firstName="" }
    name = firstName person
  do
    pure $ responseOK 
        & responseBody ?~ name
```

#### Return a list of strings as json

```haskell
{-# LANGUAGE OverloadedStrings #-}
module Main where

import AWSLambda.Events.APIGateway
import Control.Lens
import Data.Text                   (Text)
import Data.Aeson.Embedded

main :: IO ()
main = apiGatewayMain returnStringList

returnStringList :: APIGatewayProxyRequest Text -> IO (APIGatewayProxyResponse (Embedded [String]))
returnStringList request = do
  pure $ responseOK 
      & responseBodyEmbedded ?~ ["Garlands", "Head over Heels", "Treasure", "Victorialand"]
```

#### Return a data type as json

```haskell
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE DeriveGeneric #-}
module Main where

import AWSLambda.Events.APIGateway
import Control.Lens
import Data.Text                   (Text)
import Data.Aeson
import Data.Aeson.Embedded
import GHC.Generics

main :: IO ()
main = apiGatewayMain returnDataJson

data Person = Person { firstName :: String } deriving (Show, Generic)

instance FromJSON Person
instance ToJSON Person

returnDataJson :: APIGatewayProxyRequest Text -> IO (APIGatewayProxyResponse (Embedded Person))
returnDataJson request = do
  pure $ responseOK 
      & responseBodyEmbedded ?~ Person { firstName="Bobby" }
```

#### Retrieve query var and display it

(Requires `bytestring`)

```haskell
{-# LANGUAGE OverloadedStrings #-}
module Main where

import           AWSLambda.Events.APIGateway
import qualified Data.ByteString as BS
import           Data.Text.Encoding          (decodeUtf8)
import           Control.Lens
import           Data.Text                   (Text)

main :: IO ()
main = apiGatewayMain displayQueryVar

displayQueryVar :: APIGatewayProxyRequest Text -> IO (APIGatewayProxyResponse Text)
displayQueryVar request = do
  let
    params = request ^. agprqQueryStringParameters
    names = params & filter (\(x, _) -> x == "name") & map getParamValue
  do
    case names of
        [] -> return $ responseBadRequest & responseBody ?~ "Missing name argument"
        [""] -> return $ responseBadRequest & responseBody ?~ "Name argument is empty"
        [name] -> return $ responseOK & responseBody ?~ "Hello " <> (decodeUtf8 name)


getParamValue :: (BS.ByteString, Maybe BS.ByteString) -> BS.ByteString
getParamValue (_, Just value) = value
getParamValue (_, Nothing)    = ""
```

#### Get header from request and display in response

```haskell
{-# LANGUAGE OverloadedStrings #-}
module Main where

import AWSLambda.Events.APIGateway
import Data.Text.Encoding          (decodeUtf8)
import Control.Lens
import Data.Text                   (Text)

main :: IO ()
main = apiGatewayMain returnAndShowHeader

returnAndShowHeader :: APIGatewayProxyRequest Text -> IO (APIGatewayProxyResponse Text)
returnAndShowHeader request = do
  let
    headers = request ^. agprqHeaders
    headers' = filter (\(x, _) -> x == "Authorization") headers
    headerValue = headers' & head & snd & decodeUtf8
  do
    return $ responseBadRequest 
        & responseBody ?~ "Header value is: " <> headerValue
```

#### Returning status code 401 or 200 if correct header token was passed

(Requires `bytestring`)

```haskell
{-# LANGUAGE OverloadedStrings #-}
module Main where

import           AWSLambda.Events.APIGateway
import qualified Data.ByteString as BS
import           Data.ByteString             (isPrefixOf, stripPrefix)
import           Control.Lens
import           Data.Text                   (Text)
import           Data.Aeson

main :: IO ()
main = apiGatewayMain authHandler

authHandler :: APIGatewayProxyRequest Text -> IO (APIGatewayProxyResponse Text)
authHandler request = do
  let
    headers = request ^. agprqHeaders & filter (\(x, _) -> x == "Authorization")
    isAutenticated =
        case headers of
            [] -> False
            [(_, value)] -> getTokenFromHeader value & checkAuth
  do
    case isAutenticated of
        True -> return $ responseBadRequest & responseBody ?~ "Autenticated"
        False -> return $ response 401 & responseBody ?~ "Permission denied"

getTokenFromHeader :: BS.ByteString -> BS.ByteString
getTokenFromHeader token
    | "Token " `isPrefixOf` token  = stripPrefix' "Token " token
    | "Token: " `isPrefixOf` token = stripPrefix' "Token: " token
    | otherwise                    = ""

stripPrefix' :: BS.ByteString -> BS.ByteString -> BS.ByteString
stripPrefix' a b =
    case stripPrefix a b of
        Just value -> value
        Nothing -> ""

checkAuth :: BS.ByteString -> Bool
checkAuth token = token == "MY SECRET"
```


#### Adding a custom header to a response

```haskell
{-# LANGUAGE OverloadedStrings #-}
module Main where

import AWSLambda.Events.APIGateway
import Control.Lens
import Data.Semigroup
import Data.Text                   (Text)

main :: IO ()
main = apiGatewayMain handlerWithCustomHeader

handlerWithCustomHeader :: APIGatewayProxyRequest Text -> IO (APIGatewayProxyResponse Text)
handlerWithCustomHeader request = do
    pure $ responseOK
        & responseBody ?~ "Hello World"
        & agprsHeaders .~
            [ ("X-Random-Header", "Random value")
            ]
```

#### Retrieve path parameter and show different messages based on its value

```haskell
{-# LANGUAGE OverloadedStrings #-}
module Main where

import           AWSLambda.Events.APIGateway
import           Control.Lens
import qualified Data.HashMap.Strict         as HashMap
import           Data.Semigroup
import           Data.Text                   (Text)

main :: IO ()
main = apiGatewayMain handleRouteParam

handleRouteParam :: APIGatewayProxyRequest Text -> IO (APIGatewayProxyResponse Text)
handleRouteParam request = do
  let
    action =
        case HashMap.lookup "action" (request ^. agprqPathParameters) of
            Just value -> value
            Nothing -> ""
  do
    case action of
        "open"  -> pure $ responseOK & responseBody ?~ "Opening door"
        "close" -> pure $ responseOK & responseBody ?~ "Closing door"
        _       -> pure $ responseNotFound & responseBody ?~ "Invalid action"
```

#### Handling x-www-form-urlencoded data 

(Requires `uri-encode`)

```haskell
{-# LANGUAGE OverloadedStrings #-}
module Main where

import           AWSLambda.Events.APIGateway
import           Control.Lens
import           Data.Semigroup
import           Data.Aeson.TextValue
import           Data.Aeson.Embedded
import qualified Data.Text                    as T
import qualified Network.URI.Encode           as E

main :: IO ()
main = apiGatewayMain handleFormData

handleFormData :: APIGatewayProxyRequest T.Text -> IO (APIGatewayProxyResponse (Embedded [(T.Text, T.Text)]))
handleFormData request = do
  let
    rawBody :: T.Text
    rawBody =
        case (request ^. agprqBody) of
            Just (TextValue value) -> value
            Nothing -> ""
    formData :: [(T.Text, T.Text)]
    formData = getFormData rawBody
  do
    pure $ responseOK 
        & responseBodyEmbedded ?~ formData

getFormData :: T.Text -> [(T.Text, T.Text)]
getFormData body = do
    body
        & T.splitOn "&" 
        & map (T.splitOn "=")
        & map (map (E.decodeText))
        & map (\[x, y] -> (x, y))
```
