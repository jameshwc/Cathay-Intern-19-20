# Spring Boot 筆記

- 路徑用```@RequestMapping```來表示，例如根目錄想要顯示Hello World:
```java
@RestController
public class HelloWorld {
	@RequestMapping("/hello")
	public String hello(){
		return "Hey, Spring Boot 的 Hello World ! ";
	}
}
```

- ```@RequestMapping```是比較general的寫法，通常應該區分*get*和*post*，所以可以用```@GetMapping("/")```和```@PostMapping("/")```代替
    - 但RequestMapping可以用在Class之外，Get/PostMapping只能用在class之內
    - RequestMapping可以指定method，用法: ```@RequestMapping(value = "/", method = RequestMethod.GET)```
    
