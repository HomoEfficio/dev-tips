# Spring Filter Exemption 처리

JWT 검증을 Spring Filter를 활용해서 구현하기도 한다.

그런데 `/management/health` 처럼 Filtering 대상에서 면제(exemption)해줘야 하는 경우도 있다.

이 때는 아래와 같이 `OncePerRequestFilter`를 상속받아서 `shouldNotFilter(HttpServletRequest request)`를 재정의하면 쉽게 면제 처리할 수 있다.

```kotlin
class JwtAuthFilter(private val jwtProcessor: JwtProcessor): OncePerRequestFilter() {  // OncePerRequestFilter 상속 받아서

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val jwtFromRequest = getJwtFromRequest(request)
        SecurityContextHolder.getContext().authentication = jwtProcessor.extractAuthentication(jwtProcessor.parse(jwtFromRequest))
        filterChain.doFilter(request, response)
    }

    private fun getJwtFromRequest(request: HttpServletRequest): String {
        val header = request.getHeader("Authorization")
        if (header.isNullOrEmpty()) throw AccessDeniedException("No Authorization header exists.")

        val trimmedHeader = header.trim()
        if (!trimmedHeader.startsWith("Bearer ", true)) throw AccessDeniedException("No Bearer Token exists.")

        return trimmedHeader.substring("Bearer ".length).trim()
    }

    override fun shouldNotFilter(request: HttpServletRequest): Boolean {  // 면제 처리
        val orRequestMatcher = OrRequestMatcher(
                jwtExemptionList()
                    .map { AntPathRequestMatcher(it, HttpMethod.GET.name) }
                    .toList()
        )
        return orRequestMatcher.matches(request)
    }

    private fun jwtExemptionList(): List<String> {
        return listOf(
            "/management/health",
            "/management/info"
        )
    }
}

```


