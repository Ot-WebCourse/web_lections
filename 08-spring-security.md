Spring Security
------
Задача логіну й реєстрації користувача настільки часто зустрічається, що бібліотека *Spring* не може не містити засобів для реалізації цього функціоналу. Існує спеціальна бібліотека *Spring Security*, яка дозволяє в одному класі налаштувати права, необхідні для перегляду того чи іншого URL. Якщо користувач не має доступу до сторінки його автоматично буде перекинуто на сторінку логіну. 

Для початку роботи з бібліотекою, необхідно:

1. Видалити раніше додану стрічку `security.basic.enabled=false` в файлі конфігурації.

2. Необхідно мати клас реалізацію інтерфейсу `UserDetails` з яким вміє працювати *Spring Security*, для цього найбільше підходить уже існуючий клас `User`, до якого, окрім існуючих, необхідно буде додати декілька методів:

```java
@Entity
public class User implements UserDetails {
	...

	public String getUsername() {
		return getLogin();
	}
	
	public String getPassword() {
		return password;
	}

	public Collection<? extends GrantedAuthority> getAuthorities() {
		return AuthorityUtils.createAuthorityList("USER");
	}

	public boolean isAccountNonExpired() { return true; }
	public boolean isAccountNonLocked() { return true; }
	public boolean isCredentialsNonExpired() { return true; }
	public boolean isEnabled() { return true; }	
}
```

Значення більшість методів зрозуміла: `getUsername()` та `getPassword()` повертають логін та пароль користувача, які будуть використовуватись при авторизації користувача. 

Метод `getAuthorities()` повертає список ролей користувача. Ролі при цьому можуть бути описані довільними стрічками (наприклад ADMIN,MANAGER,ACCOUNTANT), які далі можна вказувати в анотаціях, щоб обмежети доступ до конкретного URL тільки визначеним ролям. Наприклад, форма керування користувачами повинна бути доступна тільки адміністраторам, а не всім користувачам. Зберігати ролі можна в окремій табличці й пов’язувати з користувачем відношенням багато до багатьох. В даному випадку ролей не передбачено, тому просто вважається, що кожен зареєстрований користувач має роль USER.

Інші методи передбачені для блокування профілю адміністратором або для обмеження часу дії профілю. В даному випадку цей функціонал не потрібен, тому всі методи повертають `true`. 

3. Необхідно реалізувати сервіс який буде повертати профіль користувача за логіном, або викидати виключення `UsernameNotFoundException`, якщо користувача не знайдено.

```java

@Service
public class UserSecurityService implements UserDetailsService {
	
	@Autowired
	private UserRepository usersRepo;
	
	@Override
	@Transactional
	public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
		
		User user = usersRepo.findByLogin(username);

		if(user == null) {
			throw new UsernameNotFoundException("User with login " + username + " was not found");
		}
		
		return user;
	}
}
```

4. Створити клас з спеціальними анотаціями, що налаштує *Spring Security*, назвемо його `SecurityConfig`:

```java
@Configuration
@EnableWebMvcSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private UserDetailsService userDetailsService;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable(); // вимкнути csrf захист

        http.authorizeRequests()
            .antMatchers("/", "/register").permitAll() // дозволити анонімним користувачам заходити на '/' та /register 
            .anyRequest().authenticated(); // всі інші запити потребують аутентифікації
            
        http.formLogin()
            .loginPage("/login")  // URL логіну
            .usernameParameter("login") // назва параметру з логіном на формі
            .permitAll() // дозволити всім заходити на форму логіну
            .and()
        .logout()
            .permitAll(); 
    }

    @Override
    public void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService)
            .passwordEncoder(new BCryptPasswordEncoder()); 
    }
}
```

Розглянемо конфігурацію детальніше:

Стрічка `http.csrf().disable();` вимикає захист від CSRF атак, щоб не ускладнювати додаток. А основне налаштування відбувається в наступній стрічці де вказується список URL `"/"` та `"/register"` (шаблони типу `"/reg*"` також підтримуються) та команда `permitAll()`, що означає дозволити заходити на ці URL всім користувачам, а далі `anyRequest().authenticated()` — вимагати ауторизуватись для всіх інших URL. Завжди необхідно дозволяти доступ до URL реєстрації та логіну всім користувачам, інакше вони просто не зможуть увійти до системи. 

Наступний блок налаштовує роботу форми логіну: вказується POST URL на який буде відправлятись форма логіну `.loginPage("/login")`, далі вказується як на формі буде називатись параметр username, оскільки в нас він називається `login`,  а не `username`, як очікує *Spring Security*. Далі дається доступ до форми логіну та виходу всім користувачам. 

У другому методі конфігурації вказується сервіс, створений раніше, за допомогою якого бібліотека знаходитиме користувача для авторизації, та метод, за допомогою якого шифрується пароль користувача (оскільки це впливає на перевірку).

Це все, що необхідно було налаштувати. Тепер кожна сторінка окрім `"/"` чи `"/register"` переправлятиме на форму логіну, якщо користувач ще не авторизований. Для того щоб отримати поточного користувача й перевіряти чи доступ є анонімним можна скористатись статичними методами, які для зручності можна додати в клас `User`:

```java
public class User implements UserDetails {
	...

	public static User getCurrentUser() {
		return (User) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
	}
	
	public static Long getCurrentUserId() {
		return getCurrentUser().getId();
	}
	
	public static boolean isAnonymous() {
		return "anonymousUser".equals(SecurityContextHolder.getContext().getAuthentication().getName());
	}
}
```

Тепер створення та отримання постів для поточного користувача матиме вигляд:

```java
@Service
public class PostsService {

	@Autowired
	private UserRepository usersRepo;
	
	@Autowired
	private PostRepository postsRepo;

	@Transactional
	public Page<Post> getPosts(int page, int pageSize) {
		User currentUser = usersRepo.findOne(User.getCurrentUserId());

		return postsRepo.findByAuthorInOrderByCreatedAtDesc(
			currentUser.getSubscriptions(), new PageRequest(page-1, pageSize));
	}

	@Transactional
	public void addPost(String text) {
		User currentUser = usersRepo.findOne(User.getCurrentUserId());

		postsRepo.save(new Post(currentUser, text, new Date()));
	}
}
```

Можливо також одразу отриматиоб’єкт `User` з сесії, однак він створений в іншій транзакції, тому безпосередньо його використовувати не можна — необхідно за ідентифікатором чи логіном отримати користувача з бази. 

