## URL Shortener with Golang and MySQL

This blog will introduce a simple but effective way to make a URL shortener like bitly.com or tinyurl.com. We can have short links in our domain for brand recognization and traffic tracking. Let's go.

# System design

**Problem**: We want to use a short link, e.g. `company.co/a3w`, instead of a long one, e.g. `company.com/?utm_source=fb&utm_campaign=blog&query=me`.

**Benefit**: Shorter link for a post, brand recognization, traffic tracking.

**Solution**: Insert the long link into database, then use its ID (integer, auto increment) as a short link. When a client gets /id, we return 308 HTTP status with the long link as redirected URL. The ID is base 10, so we can convert it to base 16 (hex) or even base 62 for shorter.

**Pros**: Simple and fast! Compared to hash method (hash of long link as ID) or random ID, we don't need any computation. Moreover, our primary key type is integer, so `insert` and `select` queries are fast.

**Cons**: Difficult to custom a short link like `company.co/liamvu`

*Notes: We can use cache for faster queries. Cache is not in the scope of this blog.*

![Screen Shot 2022-05-03 at 3.51.09 PM.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651567888486/pqi2eY-uB.png align="center")

# Setup database

This blog will use `MariaDB`, which is compatible with `MySQL`. It supports docker images running on Apple M1. We will create a database with `dbname` as name and a user with `user:pass` as username and password.

```bash
docker pull mariadb:latest
docker run -d -p 3306:3306 --name shortener-mariadb --env MARIADB_ROOT_PASSWORD=root --env MARIADB_DATABASE=dbname --env MARIADB_USER=user --env MARIADB_PASSWORD=pass mariadb:latest
```

*Tips: Use `docker ps` to check if `shortener-mariadb` is running.*

# Golang server

This section will explain core functions. The full version can be found at: https://github.com/liamhieuvu/url-shortener.git

```bash
git clone https://github.com/liamhieuvu/url-shortener.git
cd url-shortener
```

### Database connection and migration

When starting app, we connect to database with the connection created before. Then, we migrate `short_links` table using `AutoMigrate()` from [GORM](https://gorm.io). Finally, we set the start number of ID to 2001. We assume that there are 2000 short links already created. 

```go
type shortLink struct {
	ID  int64  `json:"-" gorm:"primaryKey"`
	URL string `json:"url" binding:"required,url"`
}

func main() {
	dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
	db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{})
	db.AutoMigrate(&shortLink{})
	db.Raw("alter table short_links AUTO_INCREMENT=2001").Scan(nil)
	// ...
}
```

### Creating a short link

This blog using [go-gin](https://github.com/gin-gonic/gin) as server framework. We allow to POST to `/links` with JSON payload `{"url": "https://domain.com"}`. Then, we insert it to database and get its ID. Finally, we return the base 62 of ID using `big.NewInt(links.ID).Text(62)`.

```go
func main() {
	// ....
	r := gin.Default()
	r.POST("/links", func(c *gin.Context) {
		var links shortLink
		if err := c.ShouldBindJSON(&links); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		db.Create(&links)
		c.JSON(http.StatusOK, gin.H{"short": big.NewInt(links.ID).Text(62)})
	})
	// ....
}
```

### Access the short link

When a client processes a GET request to access the short link, we convert the ID to Base 10 using `new(big.Int).SetString(n, 62)`. Then, we get the row corresponding to the ID from database. Finally, we set the real URL to the `location` field in response header and return 308 HTTP status.

```go
func main() {
	// ....
	r.GET("/:id", func(c *gin.Context) {
		id, ok := new(big.Int).SetString(c.Param("id"), 62)
		if !ok {
			c.JSON(http.StatusBadRequest, gin.H{"error": "wrong id format"})
			return
		}
		var links shortLink
		if err := db.First(&links, id.Int64()).Error; err != nil {
			c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
			return
		}
		c.Redirect(http.StatusPermanentRedirect, links.URL)
	})
	r.Run()
}
```

# Check results

Run the app

```bash
go run main.go
```

Create a short link:

```bash
$ curl --request POST 'localhost:8080/links' --header 'Content-Type: application/json' --data-raw '{"url": "https://github.com/?click=fb"}'

# result
{"short":"wh"}
```

Access the short link using `curl`:

```bash
$ curl localhost:8080/wh

# result
<a href="https://github.com/?click=fb">Permanent Redirect</a>.
```

Access the short link using browser:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651577221495/z--GHsLow.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1651577292746/2d6t-ksm_.png align="left")

That's all. Thank you for reading my blog.