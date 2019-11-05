# 제 9장 발전적인 로그인 기능

[제 8장](Chapter8.md) 에서는 기본적인 로그인 기능을 구현해보았습니다. 그러나 최근의 웹서비스에서는 (임의로) 유저의 로그인 정보를 기록해놓고, 브라우저가 재기동한 후에도 바로 로그인할 수 있는 기능 (remember me) 를 구현하는 것이 일반화되었습니다. 이번 챕터에서는 영속적 쿠키 (*permanent cookies*) 를 사용하여 이 기능을 구현해볼 것입니다. 구체적으로, 우선 유저의 로그인 정보를 오랫동안 유지하고 기록하는 (예를 들어, Bitbucket, Github에서 사용하는 방법) 에 대해 학습해보겠습니다. 그 후, [remember me] 체크박스를 사용하여 유저가 임의로 로그인 정보를 기억하는 방법에 대해 배워보겠습니다. (예를 들면 Twitter나 Facebook에서도 사용하고 있는 기능입니다.)



또한 sample 어플리케이션의 기본적인 로그인 기능은 [제 8장](Chapter8.md) 에서 구현해놓았기 때문에, 이번 챕터를 무시하고 제 10장으로 바로 진행하여도 괜찮습니다. (마찬가지로, Account의 유효성과 패스워드의 재설정도 스킵하고 제 13장으로 바로 진행할 수도 있습니다.) 그렇다고는 해도, [remember me] 기능의 구현방법을 배우는 것은, 앞으로 개발을 할 때 매우 도움이 될 것이며, 실제로 이후 이어지는 Account의 유효성(제11장) 이나 패스워드의 재설정(제12장) 등의 고도의 기능을 구현하기 위해서라도 이번 챕터는 빼뜨릴 수 없기도 합니다. 또한 웹 상에서 [remember me] 를 구비한 로그인 폼도 있습니다만, 이번 챕터에서는 이 기능의 내부 구조 ([Computer Magic](https://www.learnenough.com/command-line-tutorial#aside-computer_magic)) 를 알게될 절호의 기회라고도 말씀드릴 수 있습니다.



## 9.1 Remember me 기능

이번 섹션에서는 유저의 로그인 기능을 브라우저를 닫은 후에서도 유효한 [remember me] 기능을 구현해볼 것입니다. 이 기능을 사용하면, 유저가 명시적으로 로그아웃을 실행하지 않는 한, 로그인 상태를 유지할 수 있게 됩니다. 또한 이번 섹션에서의 후반에서는, 이 기능을 사용할지 말지를 유저가 정하는, [remember me] 의 체크박스를 로그인 폼에 추가해볼 것 입니다.



언제나처럼 일단 토픽 브랜치를 생성하고, 작업을 진행해보도록 합시다.

`$ git checkout -b advanced-login`

### 9.1.1 기억 토큰과 암호화

[8.2](Chapter8.md#82-Login) 에서는 Rails의 `session`메소드를 사용하여 유저 ID를 저장해보았습니다. 이 정보는 브라우저를 닫으면 삭제되어버리고 맙니다. 이번 섹션에서는 세션의 영속화의 첫 번째 단계로, 기억 토큰 (*Remember token*) 을 생성하고, `cookies` 메소드를 사용하여 영속적인 cookies의 생성이나 안전성이 높은 *Remember Digest* 에 의한 토큰인증에 이 기억 토큰을 활용해보겠습니다. (토큰이란, 기밀정보입니다. 패스워드와 토큰과의 차이는, 패스워드는 유저가 생성, 관리하는 정보인 것에 반해 토큰은 컴퓨터가 생성하고 관리하는 정보입니다.)



[8.2.1](Chapter8.md#821-Log_in-Method) 에서 말씀드렸다시피, `session` 메소드에서 저장한 정보는 자동적으로 안전하게 저장됩니다만, `cookies` 메소드에 보존되는 정보는 아쉽게도 이러한 처리가 되어지지 않습니다. 특히 cookies를 영속화시키면 [세션 하이잭킹](http://ja.wikipedia.org/wiki/セッションハイジャック) 이라고 하는 공격을 받을 가능성이 있습니다. 이 공격은 기억 토큰을 가로채어 특정한 유저로 대행 로그인하는 것이빈다. cookies를 훔쳐내는 방법은 다음과 같습니다.

1. 관리가 제대로 되지 않는 네트워크를 통과하는 네트워크 패킷으로부터 [패킷 스니퍼](https://ja.wikipedia.org/wiki/LANアナライザ) 라고 하는 특수한 소프트웨어로 직접 cookies를 빼낸다.
2. 데이터베이스로부터 기억 토큰을 빼낸다.
3. [크로스 사이트 스크립팅 (XSS)](http://ja.wikipedia.org/wiki/クロスサイトスクリプティング) 을 사용한다.
4. 유저가 로그인할 때 컴퓨터나 스마트폰을 직접 조작하여 해당 액세스를 해킹한다.



[7.5](Chapter7.md#75-프로의-배포) 에서는 제일 첫 번째 문제를 방지하기 위해 [Secure Sockets Layer (SSL)](https://ja.wikipedia.org/wiki/Transport_Layer_Security) 를 사이트 전체에 적용하여 네트워크 데이터를 암호화하여 보호하고, 패킷 스니퍼로부터 해킹당하지 않도록 하였습니다. 2번째 문제의 대책으로는, 기억토큰을 그대로 데이터베이스에 저장하지 않고, 기억 토큰의 해시값을 저장하도록 하였습니다. 이것은 [6.3](Chapter6.md#63-안전한-비밀번호를-추가해보자) 에서 패스워드 그 자체를 데이터베이스에 보존하는 것 보다, 패스워드의 다이제스트를 보존한 것과 같은 개념입니다. 3번째 문제는 Rails에 의해 자동적으로 처리가 됩니다. 구체적으로는 뷰의 템플릿에서 입력한 내용을 모두 자동적으로 Escape처리 해줍니다. 4번째의 로그인 중의 컴퓨터를 물리적으로 해킹을 하는 방법에 대해서는, 시스템 측에서의 근본적인 방지수단을 마련하는 것은 어렵습니다만, 2차피해를 최소화하기 위한 대응은 가능합니다. 유저가 로그아웃해있을 때 토큰을 반드시 변경하도록하여 안전상 중요시될 정보를 표시할 때는 [*디지털 서명 (digital signature)*](https://ja.wikipedia.org/wiki/デジタル署名) 처리를 하도록 합니다.



위에서 설명한 설계나 보안상의 고려사항을 기반으로하여 다음의 방침으로 영속적인 세션을 작성해보도록 합시다.

1. 기억토큰에는 랜덤한 문자열을 생성하도록 한다.
2. 브라우저의 cookies에 토큰을 보존할 때에는 유효기간을 설정한다.
3. 토큰은 해시값으로 변경하고 나서 데이터베이스에 저장한다.
4. 브라우저의 cookies에 저장하는 유저 ID는 암호화한다.
5. 영속적인 유저 ID를 포함하는 cookies를 받는다면, 해당 ID로 데이터베이스를 검색하여 기억 토큰의 cookies가 데이터베이스 내의 해시값과 일치하는 것으로 확인한다.

위의 마지막 순서가 유저가 로그인했을 때의 순서와 비슷하다는 것에 유의해주세요. 유저 로그인에서는 메일주소를 키로하여 유저를 검색하고, 송신되어진 패스워드가 패스워드 다이제스트와 일치하는 것을 (`authenticate` 메소드를 이용하여) 확인했었습니다. 즉, 여기서의 구현은 `has_secure_password` 와 비슷한 면이 많습니다.



그러면 제일 처음으로, 필요한 `remember_digest` 속성을 유저 모델에 추가해봅시다.

![](../image/Chapter9/user_model_remember_digest.png)

위 그림의 데이터모델을 어플리케이션에 추가하기 위해선, 다음의 마이그레이션을 생성해야합니다.

`$ rails generate migration add_remember_digest_to_users remember_digest:string`

([6.3.1](Chapter6.md#631-해시화된-비밀번호) 에서의 password_digest를 추가할 때의 마이그레이션과 비교해봅시다.) 전번 마이그레이션과 마찬가지로, 이번 마이그레이션 이름도 `_to_users` 로 끝나고 있습니다. 이것은 마이그레이션의 대상이 데이터베이스의 `users` 테이블이라는 것을 Rails에게 알려주기 위함입니다. 이번에는  `string` 형의 `remeber_digest` 속성을 추가하고 있습니다. 언제나처럼 Rails에 의해 디폴트의 마이그레이션이 생성됩니다.

```ruby
# db/migrate/[timestamp]_add_remember_digest_to_users.rb
class AddRememberDigestToUsers < ActiveRecord::Migration[5.0]
  def change
    add_column :users, :remember_digest, :string
  end
end
```

remember digest는 유저가 직접 읽어들일 일은 없기 때문에 (그렇게 해서도 안됩니다.) `remember_digest` 컬럼에 인덱스를 추가할 필요는 없습니다. 따라서 위 마이그레이션은 변경하지 않고 이대로 사용합니다.

`$ rails db:migrate`

여기서 기억 토큰으로서 무엇을 사용할지를 결정할 필요가 있습니다. 유력한 후보로는 여러가지를 생각할 수 있겠습니다만, 기본적으로는 길고 랜덤한 문자열이면 어떠한 것이라도 상관없습니다. 예를들어, Ruby 표준 라이브러리의 `SecureRandom` 모듈에 있는 `urlsafe_base64` 메소드라면, 이 용도에 적합할 것 같습니다. 이 메소드는 A-Z, a-z, 0-9, "-", "_" 로 이루어지는 길이 22의 랜덤한 문자열을 생성합니다. (64종류 이기때문에 `base64` 라고도 불립니다.) 전형적인 base64의 문자열은 다음과 같습니다.

```ruby
$ rails console
>> SecureRandom.urlsafe_base64
=> "q5lt38hQDc_959PVoo6b7A"
```

동일한 패스워드를 가진 유저가 여러명이 있어도 문제없는 것과 마찬가지로, 동일한 기억 토큰을 가진 유저가 여러명 있어도 문제없습니다. 세션하이잭의 리스크 등을 고려한다면, 토큰이 유니크한 성격을 가지는 것이 오히려 문제가 될 수 있습니다. 또한 방금전의 base64의 문자열에서는 64종류의 문자로부터 길이 22의 문자열이 생성되기 때문에, 2개의 기억 토큰이 가끔 완전 일치(충돌하는) ㅎ할 확률은 약 2의 -132승 정도되기 때문에, 토큰이 충돌할 일은 거의 없다고 보면 될 것입니다. 게다가 매우 고맙게도  base64는 URL을 안전하게 Escape하기 위해 사용 (`urlsafe_base64` 과 같은 이름의 메소드가 있는 것부터로도 알 수 있습니다.) 되기 때문에, base64를 채용한다면, [제 10장](Chapter10.md) 에서 Account의 유효성의 링크나 패스워드 리셋의 링크에서도 같은 토큰 생성기를 사용할 수 있게 됩니다.



유저를 기억하기 위해서는, 기억 토큰을 생성하고, 해당 토큰을 Digest로 변환한 것을 데이터베이스에 저장합니다. fixture를 테스트할 때에는 `digest` 메소드를 이미 작성해놓았기 때문에, 위 결론에 따라 새로운 토큰을 생성하기 위한 `new_token` 메소드를 생성할 수 있습니다. 이 새로운 `digest` 메소드에서는 유저 오브젝트가 불필요하기 때문에, 이 메소드도 User모델의 클래스메소드로써 생성하도록 합니다. 위 내용을 코드로 작성한 것은 아래와 같습니다.

```Ruby
# app/models/user.rb
class User < ApplicationRecord
  before_save { self.email = email.downcase }
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, presence: true, length: { minimum: 6 }

  # 넘겨진 문자열을 해시값으로 리턴합니다.
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # 랜덤한 토큰을 생성한다.
  def User.new_token
    SecureRandom.urlsafe_base64
  end
end
```

일단 기능의 구현계획으로는, `user.remember` 메소드를 생성해보도록 합시다. 이 메소드는 기억 토큰을 유저와 매칭시키고, 토큰에 대응하는 기억 Digest를 데이터베이스에 저장합니다. 마이그레이션은 실행해놓았으니, User모델에는 이미 `remember_digest` 속성이 추가되어 있습니다만, `remember_token` 속성은 아직 추가되어 있지 않습니다. 때문에 `user.remember_token` 메소드를 사용하여 토큰에 액세스할 수 있도록 하고, 토큰을 데이터베이스에 저장하지 않고 구현할 필요가 있습니다. [6.3](Chapter6.md#63-안전한-비밀번호를-추가해보자) 에서 구현한 패스워드 기능과 같은 방법으로 해결해볼 것이빈다. 그 때에는 가상의 `password` 속성과 데이터베이스 상에 있는 안전한 `password_digest` 속성, 2개를 사용하였습니다. 가상의 `password` 속성은 `has_secure_password` 메소드로 자동적으로 저장되었습니다만, 이번에는 `remember_token` 의 코드를 자신이 직접 작성할 필요가 있습니다. 이것을 작성하기 위해, [4.4.5](Chapter4.md#445-User-Class) 에서 다루어본 `attr_accessor` 를 사용하여 "가상의" 속성을 생성해보겠습니다.

```ruby
class User < ApplicationRecord
  attr_accessor :remember_token
  .
  .
  .
  def remember
    self.remember_token = ...
    update_attribute(:remember_digest, ...)
  end
end
```

`remember` 메소드의 첫 번째 행의 대입에 주목해주세요. `self` 키워드를 사용하지 않으면, Ruby에 의해 `remember_token` 이라는 이름의 *로컬 변수* 가 생성되어 버립니다. 이 동작은 Ruby에서의 오브젝트 내부에 요소대입의 스펙에 의한 것이빈다. 지금은 로컬 변수가 필요한 것이 아닙니다. `self` 키워드를 사용하면, 이 대입에 의해 유저의 `remember_token` 속성이 예상한 대로 설정될 것입니다. (6장에서 `before_save` 콜백에서 `email`이 아닌 `self.email` 이라고 기술한 것도, 이것으로 알게 되었으리라 생각합니다.) `remember` 메소드의 2번째 행에서는 `update_attribute` 메소드를 사용하여 기억 Digest를 업데이트하고 있습니다. [6.1.5](Chapter6.md#615-User-Object를-수정해보자) 에서 설명드렸다시피, 이 메소드에는 validation을 적용하지 않습니다.. 이번에는 유저의 패스워드나 패스워드 인증에 액세스하지 않기 때문에, validation을 통과시켜야만합니다.



위 고려사항들을 포함하여 유효한 토큰과, 그것에 매칭되는 Digest를 생성하도록 합니다. 구체적으로는 제일 처음에 `User.new_token` 에서 기억 토큰을 생성하고, 다음으로 `User.digest` 를 적용한 결과를 기억 Digest에 적용합니다. `remember` 메소드의 업데이트 결과는 아래와 같습니다.

```ruby
# app/models/user.rb
class User < ApplicationRecord
  attr_accessor :remember_token
  before_save { self.email = email.downcase }
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, presence: true, length: { minimum: 6 }

  # 입력받은 문자열의 해시값을 리턴한다.
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # 랜덤한 토큰을 생성하여 리턴
  def User.new_token
    SecureRandom.urlsafe_base64
  end

  # 영속적인 세션을 위해 유저를 데이터베이스에 기록한다. 
  def remember
    self.remember_token = User.new_token
    update_attribute(:remember_digest, User.digest(remember_token))
  end
end
```

##### 연습

1. 콘솔을 열고, 데이터베이스에 존재하는 제일 첫 번째 유저를 변수 `user`에 대입하여 봅시다. 그 다음, 해당 `user` 오브젝트에서 `remember` 메소드가 제대로 움직인지를 확인해봅시다. 또한,  `remember_token`과  `remember_digest` 과의 차이도 확인해봅시다.
2. 위 코드에서는 명시적으로 `User` 클래스를 호출하여 새로운 토큰이나 Digest용의 클래스메소드를 정의하였습니다. 실제로 *User.new_token* 이나 *User.digest* 를 사용하여 호출할 수 있도록 했기 때문에, 아마도 제일 명확한 클래스메소드의 정의방법이라고도 할 수 있을 것입니다. 그러나 사실은 보다 더 "Ruby적으로 올바른" 클래스 메소드의 정의방법은 2가지가 있습니다. 첫 번째는 조금 알기 복잡하며, 다른 하나는 매우 혼란스러울 것입니다. 테스트 코드를 실행하여 조금 알기 어려운 아래 첫 번째 코드의 구현이나, 매우 이해하기 혼란스러운 아래 두 번째 코드의 구현도 다 실행되는 것을 확인해보세요. *Hint* :  `self` 는 평상시의 문맥에서는 User "모델", 즉 유저 오브젝트의 *Instance* 를 지칭하고 있습니다만, 아래 코드들에서는  `self` 는 `User` "클래스" 를 지칭하고 있는 것을 알아주세요. 

```ruby
class User < ApplicationRecord
  .
  .
  .
  # 입력받은 문자열의 해시값을 리턴한다.
  def self.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # 랜덤한 토큰을 생성하여 리턴한다.
  def self.new_token
    SecureRandom.urlsafe_base64
  end
  .
  .
  .
end
```

```ruby
class User < ApplicationRecord
  .
  .
  .
  class << self
    # 입력받은 문자열의 해시값을 리턴한다.
    def digest(string)
      cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                    BCrypt::Engine.cost
      BCrypt::Password.create(string, cost: cost)
    end

    # 랜덤한 토큰을 생성하여 리턴한다.
    def new_token
      SecureRandom.urlsafe_base64
    end
  end
  .
  .
```

### 9.1.2 로그인 상태의 저장

`user.remember` 메소드가 동작할 수 있게 되었습니다. 유저의 암호화가 끝난 ID와 기억 토큰을 브라우저의 영속적 cookies에 저장하여 영속적인 세션을 생성할 준비가 되었습니다. 이 것을 실제로 실행하기 위해 `cookies` 메소드를 사용합니다. 이 메소드는 `session` 과 비슷하게 해시로 다루어집니다. 개별적인 cookies는 하나의 `value` 와 옵션의 `expires`(유효기간) 으로 구성되어 있습니다. 유효기한은 생략가능 합니다. 예를 들어, 다음과 같이 20년 후에 기한이 끝나는 기억 토큰과 같은 값을 cookies에 저장하여, 영속적인 세션을 생성할 수 있습니다.

`cookies[:remember_token] = {value: remember_token, expires: 20.years.from_now.utc}`

(위 코드에서는 Rails의 편리한 time헬퍼도 사용하고 있습니다. 상세한 내용은 [컬럼 9.1](#컬럼91-cookies는-지금부터-20년후에-만료된다-20.years.from_now) 을 확인해주세요.) 이렇게 20년 후에 끝나는 cookies설정은 자주 쓰이기에, 지금은 Rails에서도 `permanent` 라 하는 전용 메소드가 추가되었을 정도입니다. 이 메소드를 사용하면 코드는 다음과 같이 매우 심플하게 바뀝니다.

`cookies.permanent[:remember_token] = remember_token`

위 메소드에 의해서 Rails는 기한을 `20.years.from_now` 로 설정합니다.

###### 컬럼 9.1 cookies는 지금으로부터 20년후에 만료된다(20.years.from_now)

> Ruby의 기본 클래스들을 포함한 모든 클래스에 메소드를 추가할 수 있다는 것을 [4.4.2](Chapter4.md#442-Class의-상속) 에서 학습했습니다. 그 때는 `palindrome?` 이라는 메소드를 String 클래스에 추가했습니다. (겸사겸사 "deified" 도 거꾸로 표현할 수 있는 문자라는 것을 알게 되었습니다.) 또한, Rails가 사실은 `blank?` 메소드를 Object 클래스에 추가하고 있는 것도 알게 되었습니다. (이것으로, `"".blank?`, `" ".blank?`, `nil.blank?` 는 어떤 것도 `true`를 리턴합니다. 이 `cookies.permanent` 메소드에서는, cookies가 20년후에 만료된다는 (`20.years.from_now`) 지정을 할 수 있습니다만, 이것은 Rails의 *time* 헬퍼를 사용한 예제입니다. time헬퍼는 Rails에 의해 수치관련된 기저클래스인 `Fixnum` 클래스에 추가되어 있습니다.
>
> ``` ruby
>  $ rails console
>   >> 1.year.from_now
>   => Wed, 21 Jun 2017 19:36:29 UTC +00:00
>   >> 10.weeks.ago
>   => Tue, 12 Apr 2016 19:36:44 UTC +00:00
> ```
>
> Rails는 다음과 같은 헬퍼도 가지고 있습니다.
>
>  ```ruby
>   >> 1.kilobyte
>   => 1024
>   >> 5.megabytes
>   => 5242880
>  ```
>
> 위 헬퍼는 파일의 업로드에 `5.megabytes` 등의 제한을 부여할 때 편리합니다.
>
>  
>
>  메소드를 기본 클래스에 추가할 수 있는 높은 유연성덕분에, 순수한 Ruby를 자연스럽게 확장시키는 것도 가능합니다. (물론 주의할 필요는 있습니다.) 실제로 Rails의 멋지고 우아한 스펙의 많은 부분은, Ruby의 높은 확장성에 의해 실현된 것들입니다.

유저 ID를 cookies에 저장하기 위해서는, `session`  메소드를 사용한 것과 같은 패턴을 사용합니다. 구체적으로는 아래와 같습니다.

`cookies[:user_id] = user.id`

그러 이대로는 ID가 ID 그 자체로의 텍스트로써 cookies에 저장되기 때문에, 어플리케이션의 cookies의 형식이 외부로 노출되기 쉬워져, 공격자에 의한 ID 유출이 매우 쉬워집니다. 이것을 피하기 위해 *서명이 달려있는* cookies를 사용합니다. 이 것은 cookies를 브라우저에 저장하기 전에 안전하게 암호화하는 절차입니다.

`cookies.signed[:user_id] = user.id`

cookies를 설정하면, 이 이후 페이지의 뷰에서 아래와 같이 cookies로부터 유저 정보를 습득할 수 있습니다.

`User.find_by(id: cookies.signed[:user_id])`

`cookies.signed[:user_id]` 에서는 자동적으로 유저 ID의 cookies를 복호화시켜 원래대로 되돌립니다. 이어서 bcrypt를 사용하여 `cookies[:remember_token]` 이 `remember_digest` 와 일치하는 것을 확인해봅시다. 한편, 서명된 유저 ID가 있다면, 기억토큰은 불필요한것이 아니냐는 의문을 가질 수도 있습니다. 그러나 기억 토큰이 없다면, 암호화된 ID를 습득한 공격자는, 암호화 ID를 그냥 그대로 사용하여 로그인해버릴 수도 있습니다. 현재의 설계상으로는 공격자가 설사 양쪽의 cookies를 습득한다고 해도, 진짜 유저가 로그아웃해버리면 로그인할 수 없도록 되어져 있습니다.



드디어 마지막입니다. 넘겨진 토큰이 유저의 remember Digest와 일치하는 것을 확인합니다. 이 일치 작업을 bcrypt에서 확인하기 위한 여러가지 방법이 있습니다. [secure_password의 소스코드](https://github.com/rails/rails/blob/master/activemodel/lib/active_model/secure_password.rb) 를 확인해보면 다음과 같은 비교를 행하는 코드가 있습니다.

```ruby
BCrypt::Password.new(password_digest) == unencrypted_password
```

우리의 경우, 위 코드를 참고하여 아래와 같은 코드를 사용해봅시다.

```ruby
BCrypt::Password.new(remember_digest) == remember_token
```

이 코드에 대해 조사해보면, 신기하게 구현되어 있습니다. bcrypt로 암호화된 패스워드를, 토큰과 직접 비교합니다. 즉, ==으로 비교할 때, Digest를 *암호화* 하고 있는 것일까요? 그렇지만 bcrypt의 해시는 복호화할 수 없을 것이기에, 복호화하고 있는 것은 아닙니다. [bcrypt gem의 소스코드](https://github.com/codahale/bcrypt-ruby/blob/master/lib/bcrypt/password.rb)  를 자세히 확인해보면, 비교를 하고 있는 `==` 연산자가 재정의되어있습니다. 실제로 비교를 코드로 나타낸다면 다음과 같이 되어 있습니다.

```ruby
BCrypt::Password.new(remember_digest).is_password?(remember_token)
```

실제 비교 동작은 `==` 대신 `is_password?` 라고 하는 논리값을 다루는 메소드가 행하고 있습니다. 이것으로 조금은 알게 되셨나요? 지금부터 작성할 코드에서도 이것과 비슷한 방법으로 해보겠습니다.



위 설명을 바탕으로, Remember toekn과 Remember Digest를 비교하는 `authenticated?` 메소드를 User모델 안에 정의하면 되지않을까 하고 추측해볼 수 있을 것입니다. 이 메소드는  `has_secure_password` 가 제공하는 유저 인증용의 `authenticate` 메소드와 매우 닮아 있습니다. 이 메소드의 구현결과는 아래와 같습니다. 이 `authenticated?` 메소드는 Remember Digest와 매우 끈끈하게 묶여있습니다만, 사실 다른 여러가지 용도로도 응용해볼 수 있습니다. [제 10장](Chapter10.md) 에서 이 메소드를 일반화해볼 것입니다.

```ruby
# app/models/user.rb
class User < ApplicationRecord
  attr_accessor :remember_token
  before_save { self.email = email.downcase }
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, presence: true, length: { minimum: 6 }

  # 입력받은 문자열의 해시값을 리턴한다.
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # 랜덤한 토큰을 리턴한다.
  def User.new_token
    SecureRandom.urlsafe_base64
  end

  # 영속적인 세션을 위해 유저를 데이터베이스로부터 검색한다. 
  def remember
    self.remember_token = User.new_token
    update_attribute(:remember_digest, User.digest(remember_token))
  end

  # 입력받은 토큰이 Digest와 일치하면 true를 리턴한다. 
  def authenticated?(remember_token)
    BCrypt::Password.new(remember_digest).is_password?(remember_token)
  end
end
```

`authenticated?` 메소드의 로컬변수로서 정의한 `remember_token` 은, 위 모델 코드의 `attr_accessor :remember_token` 에서 정의한 접근자와는 다르다는 것을 주의해주세요. 이번 경우에는 `is_password?`의 파라미터는 메소드 내부의 로컬변수를 참조하고 있습니다. 또한 `remember_digest` 의 속성의 쓰임새에 대해서도 주목해주세요. 이것은  `self.remember_digest` 와 마찬가지이며, 즉 [제 6장](Chapter6.md) 의 `name` 이나 `email` 을 사용하던 방법과 같은 것입니다. 실제로 `remember_digest` 의 속성은 데이터베이스의 컬럼에 대응하고 있기 때문에, Active Record에 의해 간단하게 검색하거나 저장할 수도 있습니다.



이 것으로 로그인한 유저를 기억하는 처리의 준비를 끝냈습니다. `remember` 헬퍼 메소드를 추가하여 `log_in` 과 연계시켜봅시다.

```ruby
# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      log_in user
      remember user #추가한 부분
      redirect_to user
    else
      flash.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end

  def destroy
    log_out
    redirect_to root_url
  end
end
```

`log_in` 과 마찬가지로, 위 코드에서는 실제로 Sessions 헬퍼의 동작은 `remember` 메소드의 속성인 `user.remember` 를 호출할 때까지 연기되며, 여기서 Remember token을 사용하여 생성한 토큰의 Digest를 데이터베이스에 저장합니다. 이어서 위와 마찬가지로, `cookies` 메소드로 유저 ID와 Remember token의 영속적인 `cookies`를 작성해봅시다. 작성한 코드는 아래와 같습니다.

```ruby
# app/helpers/sessions_helper.rb
module SessionsHelper

  # 입력받은 유저로 로그인합니다.
  def log_in(user)
    session[:user_id] = user.id
  end

  # 유저의 세션을 영속화합니다. 
  def remember(user)
    user.remember
    cookies.permanent.signed[:user_id] = user.id
    cookies.permanent[:remember_token] = user.remember_token
  end

  # 현재 로그인해있는 유저를 리턴합니다. 
  def current_user
    if session[:user_id]
      @current_user ||= User.find_by(id: session[:user_id])
    end
  end

  # 유저가 로그인해있으면 true, 그렇지 않다면 false를 리턴합니다. 
  def logged_in?
    !current_user.nil?
  end

  # 현재 유저를 로그아웃합니다.
  def log_out
    session.delete(:user_id)
    @current_user = nil
  end
end
```

위 코드에서는 로그인하는 유저는 브라우저에서 유효한 Remember token을 얻을 수 있게금 저장됩니다만, `current_user` 메소드에서는 일시적인 세션만 다루고 있기 때문에, 이대로라면 제대로 움직이진 않을 것입니다.

```
@current_user ||= User.find_by(id: session[:user_id])
```

영속적인 세션의 경우, `session[:user_id]` 가 존재한다면, 일시적인 세션으로부터 유저를 검색해내고, 그 이외의 경우에는 `cookies[:user_id]` 로부터 유저를 검색해내어 대응하는 영속적 세션에 로그인할 필요가 있습니다. 이것을 구현하기 위해서는 아래와 같이 코드를 작성합니다.

```ruby
if session[:user_id]
  @current_user ||= User.find_by(id: session[:user_id])
elsif cookies.signed[:user_id]
  user = User.find_by(id: cookies.signed[:user_id])
  if user && user.authenticated?(cookies[:remember_token])
    log_in user
    @current_user = user
  end
end
```

위 코드에서도  `user && user.authenticated` 를 사용하는 점에 주목해주세요. 이 코드도 동작은 합니다만, 지금 이대로라면 `session` 메소드도 `cookies` 메소드도 각각 2번씩 사용되기 때문에 불필요한 호출을 하고 있습니다. 이 것을 해결하기 위해 아래와 같이 로컬변수를 사용합니다.

```ruby
if (user_id = session[:user_id])
  @current_user ||= User.find_by(id: user_id)
elsif (user_id = cookies.signed[:user_id])
  user = User.find_by(id: user_id)
  if user && user.authenticated?(cookies[:remember_token])
    log_in user
    @current_user = user
  end
end
```

위 코드에서는 다음과 같은 구조가 사용되고 있습니다만, 조금은 이해하기 어려울 수도 있습니다.

`if (user_id = session[:user_id])`

위 코드는 비교를 하고 있는 것 처럼 보입니다만, 이것은 비교를 하고 있지 않습니다. 비교라고 한다면 `==` 를 써야합니다만, 여기에서는 *대입* 을 하고 있습니다. 이 코드를 말로 설명하자면, "유저ID가 유저 ID의 세션과 같다면.." 이 아닌, "**유저 ID에 유저 ID의 세션을 대입한 결과**, 유저 ID의 세션이 존재한다면" 이 됩니다.



앞서 설명했듯이 `current_user` 헬퍼를 정의하면, 아래와 같이 됩니다.

```ruby
# app/helpers/sessions_helper.rb
module SessionsHelper

  # 입력받은 유저로 로그인합니다.
  def log_in(user)
    session[:user_id] = user.id
  end

  # 유저의 세션을 영속화합니다. 
  def remember(user)
    user.remember
    cookies.permanent.signed[:user_id] = user.id
    cookies.permanent[:remember_token] = user.remember_token
  end

  # Remember token cookie에 대응하는 유저를 리턴합니다.
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif (user_id = cookies.signed[:user_id])
      user = User.find_by(id: user_id)
      if user && user.authenticated?(cookies[:remember_token])
        log_in user
        @current_user = user
      end
    end
  end

  # 유저가 로그인해있으면 true, 그렇지 않다면 false를 리턴합니다. 
  def logged_in?
    !current_user.nil?
  end

  # 현재 유저를 로그아웃합니다.
  def log_out
    session.delete(:user_id)
    @current_user = nil
  end
end
```

위 코드에서는 새롭게 로그인한 유저는 제대로 기록됩니다. 실제로 로그인하고나서 브라우저를 닫고, 어플리케이션을 재실행하고나서 다시 브라우저로 어플리케이션에 접속해보면, 기대했던 것 처럼 동작하는 것을 확인할 수 있습니다. 브라우저의  cookies를 브라우저에서 직접 확인해볼 수도 있습니다.

![](../image/Chapter9/cookie_in_browser_chrome.png)

어플리케이션에 현재 남겨진 문제는 이제 하나 남았습니다. 브라우저의 cookies를 삭제하는 방법이 아직 없습니다. (20년 후에는 삭제됩니다만) 유저가 로그아웃할 수 없습니다. 이것은 당연하게도 테스트 코드를 실행하면 확인할 수 있는 문제이며, 테스트 실행 결과는 실패할 것입니다.

##### 연습

1. 브라우저의 cookies를 알아보고, 로그인 후의 브라우저에서는 `remember_token` 과 암호화되어진 `user_id` 가 있는 것을 확인해봅시다.
2. 콘솔을 열고 `authenticated?` 메소드가 제대로 동작하는지 확인해봅시다.