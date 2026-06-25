# E-commerce-application-
// Add these imports
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.jpa.repository.Modifying;

// Fix CartRepository
interface CartRepository extends JpaRepository<CartItem, Long> {
    List<CartItem> findBySessionId(String sessionId);
    
    @Modifying
    @Transactional
    void deleteBySessionId(String sessionId);
}

// Complete ProductService + add the rest
@Service
class ProductService {
    @Autowired private ProductRepository productRepo;
    
    public List<Product> findAll() { return productRepo.findAll(); }
    public Product findById(Long id) { 
        return productRepo.findById(id).orElseThrow(() -> new RuntimeException("Product not found")); 
    }
    public Product save(Product product) { return productRepo.save(product); }
}

@Service
class CartService {
    @Autowired private CartRepository cartRepo;
    @Autowired private ProductRepository productRepo;
    
    public CartItem addItem(Long productId, Integer quantity, String sessionId) {
        Product product = productRepo.findById(productId)
            .orElseThrow(() -> new RuntimeException("Product not found"));
        CartItem item = new CartItem();
        item.setProduct(product);
        item.setQuantity(quantity);
        item.setSessionId(sessionId);
        return cartRepo.save(item);
    }
    
    public List<CartItem> getCart(String sessionId) {
        return cartRepo.findBySessionId(sessionId);
    }
    
    public void clearCart(String sessionId) {
        cartRepo.deleteBySessionId(sessionId);
    }
}

@Service
class PaymentService {
    @Autowired private CartRepository cartRepo;
    @Autowired private OrderRepository orderRepo;
    
    public Order processPayment(String sessionId) {
        List<CartItem> items = cartRepo.findBySessionId(sessionId);
        if (items.isEmpty()) throw new RuntimeException("Cart is empty");
        
        double total = items.stream()
            .mapToDouble(item -> item.getProduct().getPrice() * item.getQuantity())
            .sum();
        
        Order order = new Order();
        order.setSessionId(sessionId);
        order.setTotalAmount(total);
        order.setStatus("PAID");
        
        cartRepo.deleteBySessionId(sessionId);
        return orderRepo.save(order);
    }
}

// REST APIs - this is what makes it actually work
@RestController
@RequestMapping("/api/products")
class ProductController {
    @Autowired private ProductService productService;
    
    @GetMapping
    public List<Product> getAllProducts() { return productService.findAll(); }
    
    @PostMapping
    public Product addProduct(@RequestBody Product product) { return productService.save(product); }
}

@RestController
@RequestMapping("/api/cart")
class CartController {
    @Autowired private CartService cartService;
    
    @PostMapping("/add")
    public CartItem addToCart(@RequestParam Long productId, 
                              @RequestParam Integer quantity,
                              @RequestParam String sessionId) {
        return cartService.addItem(productId, quantity, sessionId);
    }
    
    @GetMapping("/{sessionId}")
    public List<CartItem> viewCart(@PathVariable String sessionId) {
        return cartService.getCart(sessionId);
    }
}

@RestController
@RequestMapping("/api/payment")
class PaymentController {
    @Autowired private PaymentService paymentService;
    
    @PostMapping("/checkout/{sessionId}")
    public Order checkout(@PathVariable String sessionId) {
        return paymentService.processPayment(sessionId);
    }
}
