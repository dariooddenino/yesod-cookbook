This example shows how to upload image files to the server and manage the uploads in a database. Each image can be deleted as well.

```haskell
#!/usr/bin/env stack
{- stack
     --resolver lts-8.17
     --install-ghc
     runghc
     --package yesod
     --package yesod-static
     --package persistent-sqlite
 -}

{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE TemplateHaskell #-}
{-# LANGUAGE QuasiQuotes #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE MultiParamTypeClasses #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE GADTs #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE ViewPatterns #-}

import Yesod
import Yesod.Static
import Data.Time (UTCTime)
import System.FilePath
import System.Directory (removeFile, doesFileExist, createDirectoryIfMissing)
import Control.Applicative ((<$>), (<*>))
import Control.Monad.Logger (runStdoutLoggingT)
import Data.Conduit
import Data.Text (unpack)
import qualified Data.ByteString.Lazy as DBL
import Data.Conduit.List (consume)
import Database.Persist
import Database.Persist.Sqlite
import Data.Time (getCurrentTime)

share [mkPersist sqlSettings,mkMigrate "migrateAll"] [persistUpperCase|
Image
    filename String
    description Textarea Maybe
    date UTCTime
    deriving Show
|]

data App = App
    { getStatic :: Static -- ^ Settings for static file serving.
    , connPool  :: ConnectionPool
    }

mkYesod "App" [parseRoutes|
/ ImagesR GET POST
/image/#ImageId ImageR DELETE
/static StaticR Static getStatic
|]

instance Yesod App where
    maximumContentLength _ (Just ImagesR) = Just $ 200 * 1024 * 1024 -- 200 megabytes
    maximumContentLength _ _ = Just $ 10 * 1024 * 1024 -- 10 megabytes

instance YesodPersist App where
    type YesodPersistBackend App = SqlBackend
    runDB action = do
        App _ pool <- getYesod
        runSqlPool action pool

instance RenderMessage App FormMessage where
    renderMessage _ _ = defaultFormMessage

uploadDirectory :: FilePath
uploadDirectory = "static"

uploadForm :: Html -> MForm Handler (FormResult (FileInfo, Maybe Textarea, UTCTime), Widget)
uploadForm = renderBootstrap $ (,,)
    <$> fileAFormReq "Image file"
    <*> aopt textareaField "Image description" Nothing
    <*> lift (liftIO getCurrentTime)

addStyle :: Widget
addStyle = do
    -- Twitter Bootstrap
    addStylesheetRemote "http://netdna.bootstrapcdn.com/twitter-bootstrap/2.1.0/css/bootstrap-combined.min.css"
    -- message style
    toWidget [lucius|.message { padding: 10px 0; background: #ffffed; } |]
    -- jQuery
    addScriptRemote "http://ajax.googleapis.com/ajax/libs/jquery/1.8.0/jquery.min.js"
    -- delete function
    toWidget [julius|
$(function(){
    function confirmDelete(link) {
        if (confirm("Are you sure you want to delete this image?")) {
            deleteImage(link);
        };
    }
    function deleteImage(link) {
        $.ajax({
            type: "DELETE",
            url: link.attr("data-img-url"),
        }).done(function(msg) {
            var table = link.closest("table");
            link.closest("tr").remove();
            var rowCount = $("td", table).length;
            if (rowCount === 0) {
                table.remove();
            }
        });
    }
    $("a.delete").click(function() {
        confirmDelete($(this));
        return false;
    });
});
|]

getImagesR :: Handler Html
getImagesR = do
    ((_, widget), enctype) <- runFormPost uploadForm
    images <- runDB $ selectList [ImageFilename !=. ""] [Desc ImageDate]
    mmsg <- getMessage
    defaultLayout $ do
        addStyle
        [whamlet|$newline never
$maybe msg <- mmsg
    <div .message>
        <div .container>
            #{msg}
<div .container>
    <div .row>
        <h2>
            Upload new image
        <div .form-actions>
            <form method=post enctype=#{enctype}>
                ^{widget}
                <input .btn type=submit value="Upload">
        $if not $ null images
            <table .table>
                <tr>
                    <th>
                        Image
                    <th>
                        Decription
                    <th>
                        Uploaded
                    <th>
                        Action
                $forall Entity imageId image <- images
                    <tr>
                        <td>
                            <a href=#{imageFilePath $ imageFilename image}>
                                #{imageFilename image}
                        <td>
                            $maybe description <- imageDescription image
                                #{description}
                        <td>
                            #{show $ imageDate image}
                        <td>
                            <a href=# .delete data-img-url=@{ImageR imageId}>
                                delete

|]

postImagesR :: Handler Html
postImagesR = do
    ((result, widget), enctype) <- runFormPost uploadForm
    case result of
        FormSuccess (file, info, date) -> do
            -- TODO: check if image already exists
            -- save to image directory
            filename <- writeToServer file
            _ <- runDB $ insert (Image filename info date)
            setMessage "Image saved"
            redirect ImagesR
        _ -> do
            setMessage "Something went wrong"
            redirect ImagesR

deleteImageR :: ImageId -> Handler ()
deleteImageR imageId = do
    image <- runDB $ get404 imageId
    let filename = imageFilename image
        path = imageFilePath filename
    liftIO $ removeFile path
    -- only delete from database if file has been removed from server
    stillExists <- liftIO $ doesFileExist path

    case (not stillExists) of 
        False  -> redirect ImagesR
        True -> do
            runDB $ delete imageId
            setMessage "Image has been deleted."
            redirect ImagesR

writeToServer :: FileInfo -> Handler FilePath
writeToServer file = do
    let filename = unpack $ fileName file
        path = imageFilePath filename
    liftIO $ fileMove file path
    return filename

imageFilePath :: String -> FilePath
imageFilePath f = uploadDirectory </> f

openConnectionCount :: Int
openConnectionCount = 10

main :: IO ()
main = do
    pool <- runStdoutLoggingT $ createSqlitePool "images.db3" openConnectionCount
    runSqlPool (runMigration migrateAll) pool
    -- Get the static subsite, as well as the settings it is based on
    createDirectoryIfMissing True uploadDirectory
    static@(Static settings) <- static uploadDirectory
    warp 3000 $ App static pool
```
