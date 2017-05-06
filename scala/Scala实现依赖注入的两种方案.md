Cake Solution方案: 

```scala

case class User(name: String, passwd: String)





trait UserRepositoryComponent {

  val userRepository: UserRepository



  trait UserRepository {

    def authenticate(user: User): User



    def create(user: User)



    def delete(user: User)

  }

}



trait UserServiceComponent { this: UserRepositoryComponent =>

  val userService: UserService



  trait UserService {

    def authenticate(username: String, password: String): User



    def create(username: String, password: String)



    def delete(user: User)

  }

}



trait UserServiceComponentImpl extends UserServiceComponent {this: UserRepositoryComponent =>



  class FakeUserService extends UserService {

    def authenticate(username: String, password: String) = userRepository.authenticate(User(username, password))



    def delete(user: User) = userRepository.delete(user)



    def create(username: String, password: String) = userRepository.create(User(username, password))



  }

}



trait UserRepositoryComponentImpl extends UserRepositoryComponent {

  class FakeUserRepository extends UserRepository {

    def authenticate(user: User): User = {

      println("authenticating user: " + user)

      user

    }

    def create(user: User) = println("creating user: " + user)

    def delete(user: User) = println("deleting user: " + user)

  }

}



object DI extends UserServiceComponent with UserRepositoryComponentImpl with UserServiceComponentImpl {

  val userRepository: UserRepository = new FakeUserRepository

  val userService: UserService = new FakeUserService

}

```



type class方案：

```scala

case class User(name: String, passwd: String)



trait UserRepository {

  def authenticate(user: User): User



  def create(user: User)



  def delete(user: User)

}



trait UserService {

  def authenticate(username: String, password: String): User



  def create(username: String, password: String)



  def delete(user: User)

}



class FakeUserRepository extends UserRepository {

  def authenticate(user: User): User = {

    println("authenticating user: " + user)

    user

  }

  def create(user: User) = println("creating user: " + user)

  def delete(user: User) = println("deleting user: " + user)

}



class FakeUserService(env : {

  val userRepository: UserRepository

}) extends UserService {

  private val userRepository = env.userRepository



  def authenticate(username: String, password: String) = userRepository.authenticate(User(username, password))



  def delete(user: User) = userRepository.delete(user)



  def create(username: String, password: String) = userRepository.create(User(username, password))

}



object DI {

  val userRepository: UserRepository = new FakeUserRepository

  val userService: UserService = new FakeUserService(this)

}



```



Scalaz Reader:

```scala

import scalaz.Reader



case class User(name: String, passwd: String)



trait UserRepository {

  def authenticate(user: User): User



  def create(user: User)



  def delete(user: User)

}



trait UserService {

  def authenticate(username: String, password: String) = Reader((userRepo: UserRepository) => userRepo.authenticate(User(username, password)))



  def create(username: String, password: String) = Reader((userRepo: UserRepository) => userRepo.create(User(username, password)))



  def delete(user: User) = Reader((userRepo: UserRepository) => userRepo.delete(user))

}



class FakeUserRepository extends UserRepository {

  def authenticate(user: User): User = {

    println("authenticating user: " + user)

    user

  }

  def create(user: User) = println("creating user: " + user)

  def delete(user: User) = println("deleting user: " + user)

}



class Application extends UserService



object Test extends App {

  val userRepo = new FakeUserRepository



  val app = new Application



  val authResult = app.authenticate("admin", "passwd")(userRepo)



  println(authResult)

}

```

