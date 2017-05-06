build.gradle

```
task printProps << {
    println "db.url:  $config.database.url"
    println "db.user: $config.database.user"
    println "db.password:   $config.database.password"
    println "db.dbname:   $config.database.dbname"
}

def loadConfiguration() {
    def environment = hasProperty('env') ? env : 'dev'
    project.ext.environment = environment

    def configFile = file('config.groovy')
    def config = new ConfigSlurper(environment).parse(configFile.toURI().toURL())
    project.ext.config = config
}
```

同目录下，config.groovy

```
environments {
    dev {
        database {
            url = "jdbc:mysql://127.0.0.1/"
            user = "root"
            password = ""
            dbname = "diamond"
        }
    }

    test {
        database {
            url = ""
            user = ""
            password = ""
            dbname = ""
        }
    }

    prod {
        database {
            url = ""
            user = ""
            password = ""
            dbname = "bizdb"
        }
    }
}
```