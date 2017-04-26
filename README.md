# Ruby on Azure App Service 
## Notes
- Current version in production is 2.3-1
- 2.3-0 does not work with current build of App Service

## Changes from 2.3-0 - 2.3-1 
- Move startup steps from behind-the-scenes to startup.sh 
- Remove passenger from default startup
- Logs are 100% stored in LogFiles/docker (container stdout)

## About the docker image
This ruby image is built around the idea that it does not contain the site itself, rather the site will be mounted to the home/site/wwwroot directory. The site is assumed to have packaged gems in the vendor/bundle folder and gems cached in vendor/cache. These steps are taken care of in Kudu during deployment. In app service, the site exists on a shared volume and is mounted to into the container on each worker. 

## Key Points for Running Ruby Site
- Default sites run in "production" mode. Make sure your site runs in production mode locally before trying on app service. 
- Gems are installed and packaged on deployment, do not push vendored gems to the site. 

## Ruby-Specific AppSettings
These app settings are checked for and honored in ruby site deployment on app service. 
### Deployment specific appsettings
- BUNDLE_WITHOUT - in Kudu, this setting will be set as such: "bundle install --without $BUNDLE_WITHOUT"
- ASSETS_PRECOMPILE - if set, we run "bundle exec rake --trace assets:precompile" in deployment through kudu
### Site container specific appsettings
- SECRET_KEY_BASE - if not set we will set it for you
- RAILS_ENV - if not set, we default to 'production' and run it as 'rails server -e $RAILS_ENV'
- APP_COMMAND_LINE - overrides the 'rails server' command. Make sure your site starts on port 3000

## Deployment steps 
When a site is deployed to App Service using git, the Kudu site (appsvc/kudu:1.3) will run a series of steps on the site to make it deployment ready. The next release (1.4) will have extra steps which will be marked.  
1. Check for existence of Gemfile
2. run "bundle clean" to remove any changed/unnecessary gems (1.4)
3. run "bundle install --deployment $OPTIONS" to install gems to vendor folder. $OPTIONS will turn into any --without parameters. (1.4 uses --path \"vendor/bundle\" instead of --deployment)
4. run "bundle package" to package gems into vendor/cache folder (1.4) 
5. If $ASSETS_PRECOMPILE is defined as true, then we run "bundle exec rake --trace assets:precompile"

## Site startup 
If you look at the startup.sh file you will see the steps taken when the site container starts. 
1. Start ssh server on container
2. Generate $SECRET_KEY_BASE if the user hasn't provided one already (to allow running in production mode) 
3. Default $RAILS_ENV to production if not specified
4. remove any pid files from tmp/pids
5. run "bundle check" in vendor/bundle to make sure all dependencies are satisfied, if not, container fails to start (this is because gems shouldn't be installed on site start, only in deployment. So if check fails, run deployment again or install gems locally and directly drop them into the vendor folder.
6. Run "bundle install --local --deployment" just in case. If a gem doesn't exist then it will try to install the gem from the vendor/cache folder but won't try to download from the "source" defined in Gemfile. 
7. If $APP_COMMAND_LINE is not set, default to "rails server -e $RAILS_ENV", otherwise run $APP_COMMAND_LINE to start the server. 
