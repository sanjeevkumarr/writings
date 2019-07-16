Setting up a GO work space in local
GOROOT=location to you installation
GOPATH=location to your work space
the go path must contain "src" "bin" "pkg"  folders in it
Your project must be in the src folder.

All go files must have pcakge declared pacakge "sample_module"

There must be atleast on e main package in the go project.
package  "main"


to get a dependency can use 

go get -u "url"

//Check if all dependencies are fine
dep ensure -v




