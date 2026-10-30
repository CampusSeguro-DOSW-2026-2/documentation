# Diagrama de clases MS01:

```mermaid
---
title: "MS01 Identidad y Acceso: diagrama de clases (src/main)"
config:
  class:
    hideEmptyMembersBox: true
---
classDiagram
    direction LR

    %% Alcance: solo src/main/java (no se incluyen las pruebas de src/test).
    %% Se omiten getters/setters triviales, constantes (static final) y loggers.
    %% Estereotipos: <<interface>> <<enumeration>> <<record>> <<abstract>> <<utility>> o la anotacion Spring de la clase.

    class IdentityServiceApplication {
        <<utility>>
        +main(String[])$ void
    }

    class AuditConfig {
        <<Configuration>>
        +auditEventPublisher(String, String) AuditEventPublisher
    }
    class BootstrapAdminInitializer {
        <<Component>>
        -UserRepository userRepository
        -PasswordEncoder passwordEncoder
        -String email
        -String password
        -String firstName
        -String lastName
        +BootstrapAdminInitializer(UserRepository, PasswordEncoder, String, String, String, String)
        +run(ApplicationArguments) void
        +createFirstAdminIfNeeded() boolean
    }
    class JwtConfig {
        <<Configuration>>
        +clock() Clock
        +jwtService(String, String, Clock) JwtService
    }
    class PasswordRecoveryConfig {
        <<Configuration>>
        +recoveryPolicy(int, int, int) RecoveryPolicy
        +mailExecutor() Executor
    }
    class RecoveryPolicy {
        <<record>>
        +int codeTtlMinutes
        +int maxAttempts
        +int resendCooldownSeconds
        +codeTtl() Duration
        +resendCooldown() Duration
    }
    class SecurityConfig {
        <<Configuration>>
        +securityFilterChain(HttpSecurity, boolean) SecurityFilterChain
        +passwordEncoder() PasswordEncoder
    }
    class WebMvcConfig {
        <<Configuration>>
        +addArgumentResolvers(List~HandlerMethodArgumentResolver~) void
    }

    class AuthController {
        <<RestController>>
        -UserService userService
        -AuthService authService
        +AuthController(UserService, AuthService)
        +register(UserCreateRequest, HttpServletRequest) ResponseEntity~UserResponse~
        +login(LoginRequest, HttpServletRequest) ResponseEntity~LoginResponse~
    }
    class ErrorResponseController {
        <<RestController>>
        +handleError(HttpServletRequest) ResponseEntity~ErrorResponse~
    }
    class PasswordController {
        <<RestController>>
        -PasswordService passwordService
        +PasswordController(PasswordService)
        +forgot(ForgotPasswordRequest, HttpServletRequest) ResponseEntity~MessageResponse~
        +reset(ResetPasswordRequest, HttpServletRequest) ResponseEntity~MessageResponse~
        +change(RequesterIdentity, ChangePasswordRequest, HttpServletRequest) ResponseEntity~MessageResponse~
    }
    class RoleController {
        <<RestController>>
        -RoleService roleService
        +RoleController(RoleService)
        +getRoles(RequesterIdentity) ResponseEntity~List~RolePermissionsResponse~~
        +changeRoles(RequesterIdentity, ChangeRolesRequest, HttpServletRequest) ResponseEntity~ChangeRolesResponse~
    }
    class SessionController {
        <<RestController>>
        -SessionService sessionService
        +SessionController(SessionService)
        +logout(RequesterIdentity, HttpServletRequest) ResponseEntity~MessageResponse~
    }
    class UserController {
        <<RestController>>
        -UserService userService
        +UserController(UserService)
        +searchUsers(RequesterIdentity, String, String, String, String, String) ResponseEntity~PageResponse~UserProfileResponse~~
        +changeUserStatus(RequesterIdentity, ChangeStatusRequest, HttpServletRequest) ResponseEntity~ChangeStatusResponse~
        +getMyProfile(RequesterIdentity) ResponseEntity~UserProfileResponse~
        +updateMyProfile(RequesterIdentity, UserUpdateProfileRequest) ResponseEntity~UserProfileResponse~
        +getUserById(RequesterIdentity, UUID) ResponseEntity~UserProfileResponse~
    }

    class AuthService {
        <<interface>>
        +login(LoginRequest, String) LoginResponse
    }
    class AuthServiceImpl {
        <<Service>>
        -UserRepository userRepository
        -PasswordEncoder passwordEncoder
        -JwtService jwtService
        -AuditRecorder auditRecorder
        -RequestValidator requestValidator
        -String dummyHash
        +AuthServiceImpl(UserRepository, PasswordEncoder, JwtService, AuditRecorder, RequestValidator)
        +login(LoginRequest, String) LoginResponse
    }
    class ChangeRolesCommand {
        <<record>>
        +Set~UUID~ userIds
        +Set~UserRole~ rolesToAdd
        +Set~UserRole~ rolesToRemove
    }
    class ChangeRolesValidator {
        <<utility>>
        +validate(ChangeRolesRequest)$ ChangeRolesCommand
    }
    class EmailService {
        <<interface>>
        +sendRecoveryCode(String, String, int) void
    }
    class EmailServiceImpl {
        <<Service>>
        -JavaMailSender mailSender
        -String from
        +EmailServiceImpl(JavaMailSender, String)
        +sendRecoveryCode(String, String, int) void
    }
    class PasswordService {
        <<interface>>
        +requestRecoveryCode(ForgotPasswordRequest, String) void
        +resetPassword(ResetPasswordRequest, String) void
        +changePassword(RequesterIdentity, ChangePasswordRequest, String) void
    }
    class PasswordServiceImpl {
        <<Service>>
        -UserRepository userRepository
        -PasswordResetCodeRepository codeRepository
        -PasswordEncoder passwordEncoder
        -EmailService emailService
        -AuditRecorder auditRecorder
        -RequestValidator requestValidator
        -Clock clock
        -RecoveryPolicy policy
        -Executor mailExecutor
        -SecureRandom secureRandom
        -String dummyPassword
        -String dummyHash
        +PasswordServiceImpl(UserRepository, PasswordResetCodeRepository, PasswordEncoder, EmailService, AuditRecorder, RequestValidator, Clock, RecoveryPolicy, Executor)
        +requestRecoveryCode(ForgotPasswordRequest, String) void
        +resetPassword(ResetPasswordRequest, String) void
        +changePassword(RequesterIdentity, ChangePasswordRequest, String) void
    }
    class RoleChange {
        <<record>>
        +UserEntity user
        +Set~UserRole~ before
        +Set~UserRole~ after
    }
    class RoleService {
        <<interface>>
        +getRoles(RequesterIdentity) List~RolePermissionsResponse~
        +changeRoles(RequesterIdentity, ChangeRolesRequest, String) ChangeRolesResponse
    }
    class RoleServiceImpl {
        <<Service>>
        -UserRepository userRepository
        -AuditRecorder auditRecorder
        +RoleServiceImpl(UserRepository, AuditRecorder)
        +getRoles(RequesterIdentity) List~RolePermissionsResponse~
        +changeRoles(RequesterIdentity, ChangeRolesRequest, String) ChangeRolesResponse
    }
    class SessionService {
        <<interface>>
        +logout(RequesterIdentity, String) void
    }
    class SessionServiceImpl {
        <<Service>>
        -AuditRecorder auditRecorder
        +SessionServiceImpl(AuditRecorder)
        +logout(RequesterIdentity, String) void
    }
    class UserSearchCriteria {
        <<record>>
        +String email
        +UserState state
        +UserRole role
        +int page
        +int size
    }
    class UserSearchValidator {
        <<utility>>
        +validate(String, String, String, String, String)$ UserSearchCriteria
    }
    class UserService {
        <<interface>>
        +create(UserCreateRequest, String) UserResponse
        +getMyProfile(RequesterIdentity) UserProfileResponse
        +updateProfile(RequesterIdentity, UserUpdateProfileRequest) UserProfileResponse
        +getUserById(RequesterIdentity, UUID) UserProfileResponse
        +search(RequesterIdentity, String, String, String, String, String) PageResponse~UserProfileResponse~
        +changeUserStatus(RequesterIdentity, ChangeStatusRequest, String) ChangeStatusResponse
    }
    class UserServiceImpl {
        <<Service>>
        -UserRepository userRepository
        -PasswordEncoder passwordEncoder
        -AuditRecorder auditRecorder
        -RequestValidator requestValidator
        -Set~String~ institutionalDomains
        +UserServiceImpl(UserRepository, PasswordEncoder, AuditRecorder, RequestValidator, List~String~)
        +create(UserCreateRequest, String) UserResponse
        +getMyProfile(RequesterIdentity) UserProfileResponse
        +updateProfile(RequesterIdentity, UserUpdateProfileRequest) UserProfileResponse
        +getUserById(RequesterIdentity, UUID) UserProfileResponse
        +search(RequesterIdentity, String, String, String, String, String) PageResponse~UserProfileResponse~
        +changeUserStatus(RequesterIdentity, ChangeStatusRequest, String) ChangeStatusResponse
    }

    class PasswordResetCodeRepository {
        <<interface>>
        +findFirstByUserIdOrderByCreatedAtDesc(UUID) Optional~PasswordResetCodeEntity~
        +findFirstByUserIdAndUsedAtIsNullOrderByCreatedAtDesc(UUID) Optional~PasswordResetCodeEntity~
        +invalidateUnused(UUID, Instant) int
    }
    class UserRepository {
        <<interface>>
        +existsByEmail(String) boolean
        +searchUsers(String, UserState, UserRole, Pageable) Page~UserEntity~
        +findByEmail(String) Optional~UserEntity~
        +findAllByIdInOrderById(Collection~UUID~) List~UserEntity~
        +countWithRole(UserRole) long
    }

    class PasswordResetCodeEntity {
        <<Entity>>
        -UUID id
        -UUID userId
        -String codeHash
        -int attempts
        -Instant expiresAt
        -Instant usedAt
        -Instant createdAt
        +isActiveAt(Instant) boolean
    }
    class Permission {
        <<enumeration>>
        MS01_* = 9 permisos
        MS02_* = 9 permisos
        MS03_* = 4 permisos
        MS04_* = 5 permisos
        MS05_* = 9 permisos
        MS06_* = 5 permisos
        MS07_* = 4 permisos
        MS08_* = 2 permisos
        -String code
        -String description
        +grantedToEveryRole()$ Set~Permission~
        +grantedToCommunity()$ Set~Permission~
    }
    class UserEntity {
        <<Entity>>
        -UUID id
        -String firstName
        -String secondName
        -String firstLastName
        -String secondLastName
        -String email
        -String password
        -Set~UserRole~ roles
        -UserState state
        -Instant createdAt
        -Instant updatedAt
    }
    class UserRole {
        <<enumeration>>
        ESTUDIANTE
        PROFESOR
        ADMINISTRATIVO
        BRIGADISTA
        COORDINADOR
        ORGANIZADOR
        ADMIN
        AUDITOR
        -Set~Permission~ permissions
        +hasPermission(Permission) boolean
        +getPermissionCodes() List~String~
        +permissionsOf(Collection~UserRole~)$ Set~Permission~
        +sortedByName(Collection~UserRole~)$ List~UserRole~
    }
    class UserState {
        <<enumeration>>
        ACTIVO
        INACTIVO
    }

    class ChangePasswordRequest {
        -String currentPassword
        -String newPassword
        -String newPasswordConfirmation
    }
    class ChangeRolesRequest {
        -List~String~ ids
        -List~String~ agregar
        -List~String~ retirar
    }
    class ChangeRolesResponse {
        <<record>>
        +List~UserRoles~ resultados
    }
    class ChangeStatusRequest {
        -List~String~ ids
        -String estado
        +toUserIds() Set~UUID~
        +toState() UserState
    }
    class ChangeStatusResponse {
        <<record>>
        +List~IndividualChangeStateResponse~ resultados
        +fromEntity(List~IndividualChangeStateResponse~)$ ChangeStatusResponse
    }
    class ErrorDetail {
        <<record>>
        +String campo
        +String mensaje
    }
    class ErrorResponse {
        <<record>>
        +int status
        +String codigo
        +String mensaje
        +List~ErrorDetail~ errores
        +String correlationId
        +of(int, String, String, String)$ ErrorResponse
    }
    class ForgotPasswordRequest {
        -String email
    }
    class IndividualChangeStateResponse {
        <<record>>
        +UUID id
        +UserState estadoAnterior
        +UserState estadoNuevo
        +fromEntity(UserEntity, UserState)$ IndividualChangeStateResponse
    }
    class LoginRequest {
        -String correo
        -String contrasena
    }
    class LoginResponse {
        <<record>>
        +String token
        +String tipo
        +String expiraEn
        +UsuarioResumen usuario
    }
    class MessageResponse {
        <<record>>
        +String mensaje
    }
    class PageResponse~T~ {
        <<record>>
        +List~T~ contenido
        +int page
        +int size
        +long totalElementos
        +int totalPaginas
        +from(Page~T~)$ PageResponse~T~
    }
    class ResetPasswordRequest {
        -String email
        -String code
        -String newPassword
        -String newPasswordConfirmation
    }
    class RolePermissionsResponse {
        <<record>>
        +UserRole rol
        +List~String~ permisos
        +from(UserRole)$ RolePermissionsResponse
    }
    class StrictRequest {
        <<abstract>>
        -Set~String~ unknownFields
        +registerUnknownField(String, Object) void
        +unknownFieldNames() Set~String~
        +messageForUnknownField(String) String
        +unknownFieldErrors() List~ErrorDetail~
    }
    class UserCreateRequest {
        -String firstName
        -String secondName
        -String firstLastName
        -String secondLastName
        -String email
        -String password
        -String passwordConfirmation
    }
    class UserProfileResponse {
        <<record>>
        +UUID id
        +String firstName
        +String secondName
        +String firstLastName
        +String secondLastName
        +String email
        +List~UserRole~ roles
        +UserState state
        +fromEntity(UserEntity)$ UserProfileResponse
    }
    class UserResponse {
        <<record>>
        +UUID id
        +String email
        +List~UserRole~ roles
        +UserState state
        +fromEntity(UserEntity)$ UserResponse
    }
    class UserRoles {
        <<record>>
        +String id
        +List~UserRole~ roles
    }
    class UserUpdateProfileRequest {
        -String firstName
        -String secondName
        -String firstLastName
        -String secondLastName
        +messageForUnknownField(String) String
    }
    class UsuarioResumen {
        <<record>>
        +UUID id
        +String correo
        +List~UserRole~ roles
    }

    class CorrelationIdFilter {
        <<Component>>
        #doFilterInternal(HttpServletRequest, HttpServletResponse, FilterChain) void
    }
    class CorrelationIdRequest {
        -String correlationId
        ~CorrelationIdRequest(HttpServletRequest, String)
        +getHeader(String) String
        +getHeaders(String) Enumeration~String~
        +getHeaderNames() Enumeration~String~
    }
    class ErrorResponseWriter {
        <<utility>>
        +write(HttpServletRequest, HttpServletResponse, ErrorCode, String)$ void
    }
    class ForbiddenAccessDeniedHandler {
        +handle(HttpServletRequest, HttpServletResponse, AccessDeniedException) void
    }
    class IdentityHeaderParser {
        <<utility>>
        +parse(String, String)$ RequesterIdentity
    }
    class IdentityHeadersAuthenticationFilter {
        #doFilterInternal(HttpServletRequest, HttpServletResponse, FilterChain) void
    }
    class IssuedToken {
        <<record>>
        +String token
        +Instant expiresAt
    }
    class JwtService {
        <<interface>>
        +issueToken(UUID, String, List~UserRole~) IssuedToken
    }
    class JwtServiceImpl {
        -PrivateKey privateKey
        -String keyId
        -Clock clock
        +JwtServiceImpl(PrivateKey, String, Clock)
        +issueToken(UUID, String, List~UserRole~) IssuedToken
    }
    class PermissionChecker {
        <<utility>>
        +require(RequesterIdentity, Permission)$ void
    }
    class RequesterIdentity {
        <<record>>
        +UUID userId
        +Set~UserRole~ roles
        +permissions() Set~Permission~
        +hasPermission(Permission) boolean
        +rolesHeader() String
    }
    class RequesterIdentityArgumentResolver {
        +supportsParameter(MethodParameter) boolean
        +resolveArgument(MethodParameter, ModelAndViewContainer, NativeWebRequest, WebDataBinderFactory) Object
    }
    class RsaKeyUtils {
        <<utility>>
        +parsePrivateKey(String)$ PrivateKey
        +generateKeyPair()$ KeyPair
    }
    class UnauthenticatedEntryPoint {
        +commence(HttpServletRequest, HttpServletResponse, AuthenticationException) void
    }

    class FieldRules {
        <<utility>>
    }
    class FieldsMatch {
        <<annotation>>
        +message() String
        +field() String
        +matchField() String
        +groups() Class~?~[]
        +payload() Class~Payload~[]
    }
    class FieldsMatchImpl {
        -String field
        -String matchField
        -String message
        +initialize(FieldsMatch) void
        +isValid(Object, ConstraintValidatorContext) boolean
    }
    class RequestValidator {
        <<Component>>
        -Validator validator
        +RequestValidator(Validator)
        +collectErrors(Object) List~ErrorDetail~
        +validate(Object) void
    }
    class ValidationErrors {
        <<utility>>
        +fromViolations(Class~?~, Collection~ConstraintViolation~?~~)$ List~ErrorDetail~
        +fromFieldErrors(Class~?~, List~FieldError~)$ List~ErrorDetail~
        +merge(List~ErrorDetail~)$ List~ErrorDetail~
        ~wireName(Class~?~, String)$ String
    }

    class AuditAction {
        <<enumeration>>
        REGISTRO
        LOGIN
        LOGOUT
        CAMBIO_ROL
        CAMBIO_ESTADO
        CAMBIO_CONTRASENA
        RECUPERACION
    }
    class AuditEvent {
        <<record>>
        +AuditAction accion
        +AuditResult resultado
        +UUID usuarioId
        +String usuarioRoles
        +String detalle
        +String correlationId
        +Instant fecha
        +String recursoAfectado
    }
    class AuditEventPublisher {
        <<interface>>
        +publish(AuditEvent) void
    }
    class AuditEventPublisherImpl {
        -HttpClient httpClient
        -URI endpoint
        -String serviceKey
        -Duration requestTimeout
        +AuditEventPublisherImpl(HttpClient, String, String, Duration)
        +publish(AuditEvent) void
        ~toJson(AuditEvent)$ String
    }
    class AuditRecorder {
        <<Component>>
        -AuditEventPublisher publisher
        -Clock clock
        +AuditRecorder(AuditEventPublisher, Clock)
        +recordAnonymous(AuditAction, AuditResult, UUID, String, String, String) void
        +recordAs(AuditAction, AuditResult, RequesterIdentity, String, String, String) void
        +userResource(UUID)$ String
    }
    class AuditResult {
        <<enumeration>>
        EXITO
        FALLO
    }

    class EmailAlreadyExistsException {
        <<utility>>
        +EmailAlreadyExistsException()
    }
    class ErrorCode {
        <<enumeration>>
        VALIDATION
        INVALID_CODE
        UNAUTHENTICATED
        BAD_CREDENTIALS
        FORBIDDEN
        INACTIVE_ACCOUNT
        NOT_FOUND
        METHOD_NOT_ALLOWED
        CONFLICT
        UNSUPPORTED_MEDIA_TYPE
        BUSINESS_RULE
        INTERNAL_ERROR
        -int status
        -String code
        -String message
        +forStatus(int)$ ErrorCode
    }
    class ForbiddenOperationException {
        +ForbiddenOperationException(String)
    }
    class GlobalExceptionHandler {
        <<RestControllerAdvice>>
        +handleInvalidCredentials(InvalidCredentialsException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleInactiveAccount(InactiveAccountException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleInvalidResetCode(InvalidResetCodeException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleUnauthenticated(UnauthenticatedException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleForbiddenOperation(ForbiddenOperationException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleInvalidRequest(InvalidRequestException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleEmailExists(EmailAlreadyExistsException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleUserNotFound(RuntimeException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleValidation(MethodArgumentNotValidException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleUnreadableBody(HttpMessageNotReadableException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleMissingHeader(MissingRequestHeaderException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleMissingParameter(MissingServletRequestParameterException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleTypeMismatch(MethodArgumentTypeMismatchException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleMethodNotAllowed(HttpRequestMethodNotSupportedException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleUnsupportedMediaType(HttpMediaTypeNotSupportedException, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleRouteNotFound(Exception, HttpServletRequest) ResponseEntity~ErrorResponse~
        +handleUnexpected(Exception, HttpServletRequest) ResponseEntity~ErrorResponse~
    }
    class InactiveAccountException {
        <<utility>>
        +InactiveAccountException()
    }
    class InvalidCredentialsException {
        <<utility>>
        +InvalidCredentialsException()
    }
    class InvalidRequestException {
        -List~ErrorDetail~ errors
        +InvalidRequestException(String, List~ErrorDetail~)
    }
    class InvalidResetCodeException {
        <<utility>>
        +InvalidResetCodeException()
    }
    class Messages {
        <<utility>>
    }
    class UnauthenticatedException {
        +UnauthenticatedException(String)
    }
    class UserNotFoundException {
        -UUID userId
        +UserNotFoundException(UUID)
    }
    class UsersNotFoundException {
        -List~UUID~ missingIds
        +UsersNotFoundException(List~UUID~)
    }

    class CorrelationIds {
        <<utility>>
        +resolve(String)$ String
        +current(HttpServletRequest)$ String
    }
    class GlobalValues {
        <<utility>>
    }
    class MiniJson {
        <<utility>>
        +write(Object)$ String
    }
    class Texts {
        <<utility>>
        +trimToNull(String)$ String
    }

    class OncePerRequestFilter {
        <<Spring>>
    }
    class HttpServletRequestWrapper {
        <<Jakarta>>
    }
    class HandlerMethodArgumentResolver {
        <<Spring>>
    }
    class AccessDeniedHandler {
        <<Spring>>
    }
    class AuthenticationEntryPoint {
        <<Spring>>
    }
    class ErrorController {
        <<Spring>>
    }
    class ApplicationRunner {
        <<Spring>>
    }
    class WebMvcConfigurer {
        <<Spring>>
    }
    class JpaRepository~T, ID~ {
        <<Spring>>
    }
    class ConstraintValidator~A, T~ {
        <<Jakarta>>
    }
    class RuntimeException {
        <<Java>>
    }


    %% Herencia y realizacion (proyecto)
    AuthServiceImpl ..|> AuthService
    EmailServiceImpl ..|> EmailService
    PasswordServiceImpl ..|> PasswordService
    RoleServiceImpl ..|> RoleService
    SessionServiceImpl ..|> SessionService
    UserServiceImpl ..|> UserService
    AuditEventPublisherImpl ..|> AuditEventPublisher
    JwtServiceImpl ..|> JwtService
    ChangePasswordRequest --|> StrictRequest
    ChangeRolesRequest --|> StrictRequest
    ChangeStatusRequest --|> StrictRequest
    ForgotPasswordRequest --|> StrictRequest
    LoginRequest --|> StrictRequest
    ResetPasswordRequest --|> StrictRequest
    UserCreateRequest --|> StrictRequest
    UserUpdateProfileRequest --|> StrictRequest

    %% Herencia y realizacion (framework)
    CorrelationIdFilter --|> OncePerRequestFilter
    IdentityHeadersAuthenticationFilter --|> OncePerRequestFilter
    CorrelationIdRequest --|> HttpServletRequestWrapper
    RequesterIdentityArgumentResolver ..|> HandlerMethodArgumentResolver
    ForbiddenAccessDeniedHandler ..|> AccessDeniedHandler
    UnauthenticatedEntryPoint ..|> AuthenticationEntryPoint
    ErrorResponseController ..|> ErrorController
    BootstrapAdminInitializer ..|> ApplicationRunner
    WebMvcConfig ..|> WebMvcConfigurer
    UserRepository --|> JpaRepository
    PasswordResetCodeRepository --|> JpaRepository
    FieldsMatchImpl ..|> ConstraintValidator
    EmailAlreadyExistsException --|> RuntimeException
    ForbiddenOperationException --|> RuntimeException
    InactiveAccountException --|> RuntimeException
    InvalidCredentialsException --|> RuntimeException
    InvalidRequestException --|> RuntimeException
    InvalidResetCodeException --|> RuntimeException
    UnauthenticatedException --|> RuntimeException
    UserNotFoundException --|> RuntimeException
    UsersNotFoundException --|> RuntimeException

    %% Composicion de tipos anidados
    CorrelationIdFilter *-- CorrelationIdRequest
    RoleServiceImpl *-- RoleChange
    ChangeRolesResponse *-- "*" UserRoles
    LoginResponse *-- UsuarioResumen
    ErrorResponse *-- "*" ErrorDetail
    ErrorCode *-- Messages

    %% Capa web: controladores -> servicios y DTO
    AuthController --> AuthService
    AuthController --> UserService
    PasswordController --> PasswordService
    RoleController --> RoleService
    SessionController --> SessionService
    UserController --> UserService
    AuthController ..> UserCreateRequest
    AuthController ..> LoginRequest
    AuthController ..> UserResponse
    AuthController ..> LoginResponse
    PasswordController ..> ForgotPasswordRequest
    PasswordController ..> ResetPasswordRequest
    PasswordController ..> ChangePasswordRequest
    PasswordController ..> MessageResponse
    RoleController ..> ChangeRolesRequest
    RoleController ..> ChangeRolesResponse
    RoleController ..> RolePermissionsResponse
    SessionController ..> MessageResponse
    UserController ..> ChangeStatusRequest
    UserController ..> ChangeStatusResponse
    UserController ..> UserUpdateProfileRequest
    UserController ..> UserProfileResponse
    UserController ..> PageResponse

    %% Contratos de servicio -> DTO
    ChangeStatusResponse --> "*" IndividualChangeStateResponse

    %% Servicios: colaboradores inyectados
    AuthServiceImpl --> UserRepository
    PasswordServiceImpl --> UserRepository
    RoleServiceImpl --> UserRepository
    UserServiceImpl --> UserRepository
    PasswordServiceImpl --> PasswordResetCodeRepository
    PasswordServiceImpl --> EmailService
    PasswordServiceImpl --> RecoveryPolicy
    AuthServiceImpl --> JwtService
    AuthServiceImpl --> RequestValidator
    PasswordServiceImpl --> RequestValidator
    UserServiceImpl --> RequestValidator
    AuthServiceImpl --> AuditRecorder
    PasswordServiceImpl --> AuditRecorder
    RoleServiceImpl --> AuditRecorder
    SessionServiceImpl --> AuditRecorder
    UserServiceImpl --> AuditRecorder
    BootstrapAdminInitializer --> UserRepository

    %% Servicios: entidades, ayudantes y reglas
    RoleServiceImpl ..> ChangeRolesCommand
    RoleServiceImpl ..> ChangeRolesValidator
    UserServiceImpl ..> UserSearchCriteria
    UserServiceImpl ..> UserSearchValidator
    UserServiceImpl ..> PermissionChecker

    %% Persistencia y dominio
    PasswordResetCodeRepository ..> PasswordResetCodeEntity
    UserEntity --> "*" UserRole : roles
    UserEntity --> UserState : state
    UserRole --> "*" Permission
    RequesterIdentity --> "*" UserRole
    ChangeRolesCommand --> "*" UserRole

    %% Seguridad: cadena de filtros, identidad y JWT
    SecurityConfig ..> IdentityHeadersAuthenticationFilter
    SecurityConfig ..> UnauthenticatedEntryPoint
    SecurityConfig ..> ForbiddenAccessDeniedHandler
    WebMvcConfig ..> RequesterIdentityArgumentResolver
    IdentityHeadersAuthenticationFilter ..> IdentityHeaderParser
    RequesterIdentityArgumentResolver ..> IdentityHeaderParser
    IdentityHeaderParser ..> RequesterIdentity
    UnauthenticatedEntryPoint ..> ErrorResponseWriter
    ForbiddenAccessDeniedHandler ..> ErrorResponseWriter
    ErrorResponseWriter ..> MiniJson
    PermissionChecker ..> ForbiddenOperationException
    JwtService ..> IssuedToken
    JwtConfig ..> JwtServiceImpl
    JwtConfig ..> RsaKeyUtils
    JwtServiceImpl ..> MiniJson
    JwtServiceImpl ..> GlobalValues

    %% Configuracion
    AuditConfig ..> AuditEventPublisherImpl

    %% Auditoria
    AuditRecorder --> AuditEventPublisher
    AuditRecorder ..> AuditEvent
    AuditEvent --> AuditAction
    AuditEvent --> AuditResult
    AuditEventPublisher ..> AuditEvent
    AuditEventPublisherImpl ..> MiniJson

    %% Validacion
    ChangePasswordRequest ..> FieldsMatch : @FieldsMatch
    ResetPasswordRequest ..> FieldsMatch : @FieldsMatch
    UserCreateRequest ..> FieldsMatch : @FieldsMatch
    FieldsMatch ..> FieldsMatchImpl : validatedBy
    RequestValidator ..> StrictRequest
    ChangeRolesValidator ..> ChangeRolesCommand
    UserSearchValidator ..> UserSearchCriteria
    InvalidRequestException --> "*" ErrorDetail

    %% Manejo global de errores
    GlobalExceptionHandler ..> ErrorResponse
    ErrorResponseController ..> ErrorResponse

    %% Tipos transversales: se anotan en lugar de dibujar decenas de aristas
    note for CorrelationIds "Lo usan los controladores, los servicios,<br/>CorrelationIdFilter, ErrorResponseWriter,<br/>GlobalExceptionHandler y el publicador de auditoria<br/>para leer o generar X-Correlation-ID"
    note for RequesterIdentity "Lo reciben como parametro los controladores protegidos<br/>(Password, Role, Session, User) y sus servicios;<br/>lo inyecta RequesterIdentityArgumentResolver"
    note for ErrorCode "Lo emplean servicios, validadores, manejadores<br/>de seguridad y de errores para construir ErrorResponse"
    note for IdentityServiceApplication "@SpringBootApplication: punto de arranque<br/>que escanea todos los paquetes del servicio"
    note for UserController "Los DTO de entrada y salida aparecen en las firmas de los<br/>controladores y de las interfaces de servicio; solo se dibujan<br/>las dependencias controlador a DTO para no saturar el diagrama"
    note for Texts "Lo usan los DTO de entrada<br/>para normalizar cadenas (trimToNull)"
    note for FieldRules "Constantes de reglas de campo usadas por<br/>los DTO de entrada y UserServiceImpl"
    note for UserServiceImpl "Los servicios lanzan las excepciones<br/>del paquete exception"
```
