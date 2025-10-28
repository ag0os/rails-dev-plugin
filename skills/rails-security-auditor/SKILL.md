---
name: Rails Security Auditor
description: Automatically invoked when analyzing security concerns or vulnerabilities. Triggers on mentions of "security", "vulnerability", "SQL injection", "XSS", "CSRF", "authentication", "authorization", "secure", "exploit", "attack", "mass assignment", "sanitize", "brakeman", "sensitive data", "encryption", "secret", "password". Performs security audits of Rails applications and identifies potential vulnerabilities.
allowed-tools: Read, Grep, Glob, Bash
---

# Rails Security Auditor

Comprehensive security analysis and vulnerability detection for Rails applications.

## When to Use This Skill

- Performing security audits on Rails code
- Reviewing authentication and authorization implementations
- Identifying SQL injection vulnerabilities
- Detecting XSS (Cross-Site Scripting) risks
- Analyzing mass assignment vulnerabilities
- Checking for exposed secrets or credentials
- Validating CSRF protection
- Reviewing session management
- Auditing API security

## Security Analysis Methodology

### 1. Authentication & Authorization

**Check for proper authentication**:
```ruby
# ✅ Using Devise or similar
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
end

# ❌ Rolling your own auth without security measures
def authenticate
  user = User.find_by(email: params[:email])
  if user.password == params[:password]  # Plain text comparison!
    session[:user_id] = user.id
  end
end
```

**Verify authorization with Pundit**:
```ruby
# ✅ Proper authorization
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    authorize @post
  end
end

# ❌ No authorization check
def show
  @post = Post.find(params[:id])  # Any user can access any post!
end
```

### 2. SQL Injection Prevention

**Safe query patterns**:
```ruby
# ✅ Parameterized queries
User.where("email = ?", params[:email])
User.where(email: params[:email])

# ❌ String interpolation vulnerability
User.where("email = '#{params[:email]}'")  # SQL INJECTION!

# ❌ Raw SQL without sanitization
ActiveRecord::Base.connection.execute("SELECT * FROM users WHERE id = #{params[:id]}")
```

### 3. Cross-Site Scripting (XSS) Prevention

**Safe output in views**:
```erb
<%# ✅ Auto-escaped by default %>
<%= @post.title %>

<%# ❌ Bypasses escaping - only use for trusted content %>
<%== @post.content %>
<%= raw @post.content %>
<%= @post.content.html_safe %>

<%# ✅ Sanitize user input before marking safe %>
<%= sanitize @post.content, tags: %w(p br strong em) %>
```

### 4. Mass Assignment Protection

**Strong parameters**:
```ruby
# ✅ Whitelisted attributes
def user_params
  params.require(:user).permit(:name, :email)
end

# ❌ Permits all attributes
def user_params
  params[:user]  # Mass assignment vulnerability!
end

# ❌ Using permit! (dangerous)
def user_params
  params.require(:user).permit!
end
```

### 5. CSRF Protection

**Verify CSRF protection is enabled**:
```ruby
# ✅ Default Rails protection
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end

# ❌ CSRF protection disabled
class ApiController < ApplicationController
  skip_before_action :verify_authenticity_token  # Only for stateless APIs
end
```

### 6. Sensitive Data Exposure

**Check for exposed secrets**:
```ruby
# ❌ Hardcoded credentials
database_password = "super_secret_123"
api_key = "sk_live_abc123xyz"

# ✅ Use Rails credentials
database_password = Rails.application.credentials.database_password
api_key = Rails.application.credentials.api_key

# ❌ Logging sensitive data
Rails.logger.info "User password: #{user.password}"

# ✅ Filter sensitive parameters
Rails.application.config.filter_parameters += [:password, :ssn, :credit_card]
```

**Search for exposed secrets**:
```bash
# Check for hardcoded credentials in codebase
grep -r "password.*=.*['\"]" app/
grep -r "api_key.*=.*['\"]" app/
grep -r "secret.*=.*['\"]" app/
```

### 7. Secure Session Management

```ruby
# ✅ Secure session configuration
Rails.application.config.session_store :cookie_store,
  key: '_app_session',
  secure: Rails.env.production?,  # HTTPS only in production
  httponly: true,                 # Not accessible via JavaScript
  same_site: :lax                 # CSRF protection

# Set session timeout
config.session_store :cookie_store, expire_after: 30.minutes
```

### 8. File Upload Security

```ruby
# ❌ Unrestricted file uploads
def create
  uploaded_file = params[:file]
  File.write("public/uploads/#{uploaded_file.original_filename}", uploaded_file.read)
end

# ✅ Validate file types and sanitize filenames
def create
  uploaded_file = params[:file]

  unless allowed_content_type?(uploaded_file.content_type)
    return render json: { error: "Invalid file type" }, status: :unprocessable_entity
  end

  sanitized_filename = sanitize_filename(uploaded_file.original_filename)
  # Store with ActiveStorage or similar
end

def allowed_content_type?(type)
  %w[image/jpeg image/png image/gif application/pdf].include?(type)
end
```

## Security Checklist

### Authentication
- [ ] Is a secure authentication gem used (Devise, Authlogic)?
- [ ] Are passwords properly hashed (bcrypt)?
- [ ] Is password complexity enforced?
- [ ] Is account lockout implemented after failed attempts?
- [ ] Is two-factor authentication available?

### Authorization
- [ ] Is Pundit or similar authorization framework used?
- [ ] Are all controller actions authorized?
- [ ] Are scopes properly applied to queries?
- [ ] Is multi-tenancy data isolation enforced?

### Input Validation
- [ ] Are all user inputs validated?
- [ ] Are strong parameters used consistently?
- [ ] Are file uploads restricted and validated?
- [ ] Is user input sanitized before output?

### Data Protection
- [ ] Are passwords never stored in plain text?
- [ ] Are API keys and secrets in encrypted credentials?
- [ ] Is sensitive data filtered from logs?
- [ ] Are database backups encrypted?
- [ ] Is SSL/TLS enforced in production?

### API Security
- [ ] Is API authentication required (tokens, OAuth)?
- [ ] Are rate limits implemented?
- [ ] Is CORS properly configured?
- [ ] Are API versions properly managed?

### Dependencies
- [ ] Are all gems up to date?
- [ ] Are there known vulnerabilities (bundle audit)?
- [ ] Is Dependabot or similar tool enabled?

## Security Scanning Commands

Use Bash tool to run security scans:

```bash
# Run Brakeman security scanner
bundle exec brakeman -A -q

# Check for vulnerable dependencies
bundle exec bundle-audit check --update

# Check for outdated gems with vulnerabilities
bundle exec bundle-audit check

# Check Git for committed secrets
git secrets --scan

# Review Rails security headers
curl -I https://your-app.com
```

## Common Vulnerability Patterns

### 1. Insecure Direct Object References (IDOR)

```ruby
# ❌ IDOR Vulnerability
def show
  @document = Document.find(params[:id])  # User can access any document!
end

# ✅ Scope to current user
def show
  @document = current_user.documents.find(params[:id])
end

# ✅ Use Pundit for complex authorization
def show
  @document = Document.find(params[:id])
  authorize @document
end
```

### 2. Missing Access Controls

```ruby
# ❌ Admin action without authorization
def delete_all_users
  User.delete_all
end

# ✅ Require admin authorization
def delete_all_users
  authorize :admin_panel, :manage_users?
  User.delete_all
end
```

### 3. Timing Attacks

```ruby
# ❌ Vulnerable to timing attacks
def verify_token(submitted, actual)
  submitted == actual  # String comparison can be timed
end

# ✅ Constant-time comparison
def verify_token(submitted, actual)
  ActiveSupport::SecurityUtils.secure_compare(submitted, actual)
end
```

### 4. Redirect Vulnerabilities

```ruby
# ❌ Open redirect vulnerability
def login
  redirect_to params[:return_to]  # Can redirect anywhere!
end

# ✅ Validate redirect URLs
def login
  return_to = params[:return_to]
  if return_to && return_to.starts_with?('/')
    redirect_to return_to
  else
    redirect_to root_path
  end
end
```

## Security Headers

Ensure these security headers are configured:

```ruby
# config/application.rb or middleware
config.action_dispatch.default_headers = {
  'X-Frame-Options' => 'SAMEORIGIN',
  'X-Content-Type-Options' => 'nosniff',
  'X-XSS-Protection' => '1; mode=block',
  'Strict-Transport-Security' => 'max-age=31536000; includeSubDomains',
  'Content-Security-Policy' => "default-src 'self'"
}
```

## Secure Coding Practices

### Environment Variables
```ruby
# ✅ Never commit .env files
# Add to .gitignore:
.env
.env.local
.env.production

# Use Rails credentials instead
Rails.application.credentials.secret_api_key
```

### Database Queries
```ruby
# ✅ Always use parameterized queries
User.where("status = ? AND role = ?", status, role)

# ✅ Use hash conditions
User.where(status: status, role: role)

# ❌ Never interpolate user input
User.where("status = '#{status}'")  # SQL INJECTION!
```

### API Tokens
```ruby
# ✅ Use secure random tokens
token = SecureRandom.urlsafe_base64(32)

# ❌ Weak token generation
token = Time.now.to_i.to_s  # Predictable!
```

## Output Format

Provide security analysis in this structure:

```markdown
## Security Audit Results

### Critical Vulnerabilities (Immediate Action Required)
1. [Vulnerability type] - Location: file.rb:line
   - Risk: [High/Medium/Low]
   - Description: [What's vulnerable]
   - Exploit scenario: [How it could be exploited]
   - Fix: [Specific remediation]

### Security Warnings
...

### Best Practice Recommendations
...

### Security Configuration Review
...
```

## Compliance Considerations

- **GDPR**: Data protection, right to deletion, data portability
- **PCI-DSS**: Payment card data security (if applicable)
- **HIPAA**: Healthcare data protection (if applicable)
- **SOC 2**: Security controls and monitoring

Remember: Security is not a one-time audit but an ongoing process. Regularly review and update security practices as new vulnerabilities are discovered.
