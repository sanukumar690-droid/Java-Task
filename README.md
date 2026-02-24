# 1.Entity
@Entity
@Table(name = "wallet")
public class Wallet {

    @Id
    private UUID id;

    @Column(nullable = false)
    private BigDecimal balance = BigDecimal.ZERO;

    @Version
    private Long version;
}
2.Liquibase Migration 
# Liquibase Migration
databaseChangeLog:
  - include:
      file: db/changelog/001-create-wallet-table.yaml

    # 1 create-wallet-table.yaml
    databaseChangeLog:
  - changeSet:
      id: 001
      author: sanu
      changes:
        - createTable:
            tableName: wallet
            columns:
              - column:
                  name: id
                  type: UUID
                  constraints:
                    primaryKey: true
              - column:
                  name: balance
                  type: NUMERIC(19,2)
                  defaultValue: 0
                  constraints:
                    nullable: false
              - column:
                  name: version
                  type: BIGINT
                #  Repository (Atomic Update – Concurrency Safe)
    @Repository
public interface WalletRepository extends JpaRepository<Wallet, UUID> {

    @Modifying
    @Query("""
        UPDATE Wallet w
        SET w.balance = w.balance + :amount
        WHERE w.id = :id
    """)
    int deposit(UUID id, BigDecimal amount);

    @Modifying
    @Query("""
        UPDATE Wallet w
        SET w.balance = w.balance - :amount
        WHERE w.id = :id
        AND w.balance >= :amount
    """)
    int withdraw(UUID id, BigDecimal amount);
}

# Service Layer
@Service
@RequiredArgsConstructor
public class WalletService {

    private final WalletRepository repository;

    @Transactional
    public void operate(UUID id, OperationType type, BigDecimal amount) {

        if (type == OperationType.DEPOSIT) {
            if (repository.deposit(id, amount) == 0) {
                throw new WalletNotFoundException();
            }
        }

        if (type == OperationType.WITHDRAW) {
            int updated = repository.withdraw(id, amount);
            if (updated == 0) {
                throw new InsufficientFundsException();
            }
        }
    }

    public BigDecimal getBalance(UUID id) {
        return repository.findById(id)
                .orElseThrow(WalletNotFoundException::new)
                .getBalance();
    }
}
# Controller

@RestController
@RequestMapping("/api/v1/wallet")
@RequiredArgsConstructor
public class WalletController {

    private final WalletService service;

    @PostMapping
    public ResponseEntity<Void> operate(@Valid @RequestBody WalletRequest request) {
        service.operate(
                request.walletId(),
                request.operationType(),
                request.amount()
        );
        return ResponseEntity.ok().build();
    }

    @GetMapping("/{id}")
    public ResponseEntity<WalletResponse> get(@PathVariable UUID id) {
        return ResponseEntity.ok(
                new WalletResponse(service.getBalance(id))
        );
    }
}

# Error Handling (NO 50X)
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(WalletNotFoundException.class)
    public ResponseEntity<ErrorResponse> notFound() {
        return ResponseEntity.status(404)
                .body(new ErrorResponse("WALLET_NOT_FOUND"));
    }

    @ExceptionHandler(InsufficientFundsException.class)
    public ResponseEntity<ErrorResponse> insufficient() {
        return ResponseEntity.badRequest()
                .body(new ErrorResponse("INSUFFICIENT_FUNDS"));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> generic() {
        return ResponseEntity.badRequest()
                .body(new ErrorResponse("INVALID_REQUEST"));
    }
}

# Dockerfile
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY target/wallet.jar app.jar
ENTRYPOINT ["java","-jar","app.jar"]

# docker-compose.yml

version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: wallet
      POSTGRES_USER: wallet
      POSTGRES_PASSWORD: wallet
    ports:
      - "5432:5432"

  app:
    build: .
    depends_on:
      - postgres
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/wallet
      SPRING_DATASOURCE_USERNAME: wallet
      SPRING_DATASOURCE_PASSWORD: wallet
    ports:
      - "8080:8080"

      #

    
    
    








    
