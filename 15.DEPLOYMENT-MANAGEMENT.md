15. Triển khai & Quản lý ứng dụng với NestJS
Sau khi phát triển xong ứng dụng, việc triển khai (deployment) và quản lý (management) là những bước quan trọng để đưa ứng dụng vào hoạt động và đảm bảo nó chạy ổn định, hiệu quả. Dưới đây là các khía cạnh chính:

15.1. Build & Production mode
Khi phát triển ứng dụng NestJS, bạn thường làm việc trong môi trường phát triển (development mode). Tuy nhiên, khi triển khai lên môi trường sản phẩm (production environment), bạn cần phải "build" ứng dụng để tối ưu hóa hiệu suất và kích thước.

Development mode (Chế độ phát triển):
Sử dụng TypeScript trực tiếp (thường dùng ts-node hoặc nest start --watch).
Cung cấp thông tin debug chi tiết.
Tự động tải lại khi có thay đổi mã nguồn (hot-reloading).
Không tối ưu về hiệu suất và kích thước file.
Build:
Quá trình chuyển đổi mã nguồn TypeScript sang JavaScript (ES5 hoặc ES6 tùy cấu hình).
Thường sử dụng @nestjs/cli với lệnh nest build. Lệnh này sẽ biên dịch mã nguồn từ thư mục src sang thư mục dist (mặc định).
Kết quả là các file JavaScript thuần túy, có thể được thực thi trực tiếp bằng Node.js.
Production mode (Chế độ sản phẩm):
Chạy ứng dụng từ thư mục dist (file JavaScript đã được build) bằng lệnh node dist/main.js (giả sử main.js là file khởi tạo ứng dụng).
Tối ưu hóa hiệu suất: Loại bỏ các mã nguồn chỉ dành cho phát triển, giảm kích thước file.
Tăng cường bảo mật: Thông tin debug chi tiết thường bị ẩn hoặc giới hạn.
Sử dụng biến môi trường: Các cấu hình quan trọng (như chuỗi kết nối database, khóa API) được quản lý thông qua biến môi trường (ví dụ: .env và thư viện dotenv hoặc @nestjs/config).
Ví dụ trong NestJS:

JSON

// package.json
{
  "scripts": {
    "start:dev": "nest start --watch", // Chế độ phát triển
    "build": "nest build",             // Biên dịch ra thư mục dist
    "start:prod": "node dist/main"     // Chạy ứng dụng đã build
  }
}
15.2. Logging nâng cao
Logging là một phần quan trọng của việc quản lý ứng dụng, giúp theo dõi trạng thái, phát hiện lỗi và gỡ lỗi trong môi trường sản phẩm. NestJS tích hợp sẵn module Logger, nhưng để nâng cao hơn, bạn có thể sử dụng các thư viện logging chuyên dụng.

Logger của NestJS:
Cung cấp các cấp độ log (log, error, warn, debug, verbose).
Dễ dàng tích hợp và sử dụng.
Có thể tùy chỉnh thông qua LoggerService.
Các thư viện logging phổ biến (trong Node.js/NestJS):
Winston: Một thư viện logging mạnh mẽ và linh hoạt. Cho phép bạn cấu hình nhiều "transports" (nơi lưu trữ log như console, file, database, dịch vụ log bên ngoài).
Pino: Một logger JSON cực kỳ nhanh. Lý tưởng cho các ứng dụng hiệu suất cao và khi bạn muốn gửi log đến các hệ thống quản lý log tập trung (như ELK stack, Grafana Loki).
Các tính năng nâng cao:
Cấp độ log động: Thay đổi cấp độ log tùy theo môi trường.
Ghi log vào file: Lưu trữ log vào file để phân tích sau này.
Xoay vòng log (Log rotation): Tự động tạo file log mới khi file cũ đạt kích thước nhất định hoặc sau một khoảng thời gian.
Log tập trung: Gửi log đến các dịch vụ quản lý log như ELK Stack (Elasticsearch, Logstash, Kibana), Grafana Loki, Datadog, New Relic để phân tích và giám sát.
Thêm ngữ cảnh (Context): Ghi lại các thông tin liên quan đến request (ID request, ID người dùng) để dễ dàng theo dõi.
15.3. Cấu hình Docker
Docker là một nền tảng mở cho các nhà phát triển và SysOps xây dựng, vận chuyển và chạy các ứng dụng phân tán. Nó cho phép đóng gói ứng dụng cùng tất cả các phụ thuộc của nó (mã nguồn, thư viện, cấu hình) vào một "container" độc lập và di động.

Lợi ích khi dùng Docker cho NestJS:

Môi trường nhất quán: Đảm bảo ứng dụng chạy giống nhau trên mọi môi trường (local, staging, production).
Cô lập ứng dụng: Ứng dụng chạy trong container, không ảnh hưởng đến hệ thống máy chủ và ngược lại.
Triển khai dễ dàng: Container có thể dễ dàng di chuyển và chạy trên bất kỳ hệ thống nào có Docker.
Khả năng mở rộng: Dễ dàng triển khai nhiều bản sao của ứng dụng bằng cách chạy nhiều container.
Dockerfile: Là file văn bản chứa các chỉ dẫn để xây dựng một Docker image.

Ví dụ Dockerfile cơ bản cho NestJS:
Dockerfile

# Giai đoạn Build
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install --omit=dev # Cài đặt chỉ các dependencies cần cho production

COPY . .
RUN npm run build # Chạy lệnh build NestJS

# Giai đoạn Production
FROM node:20-alpine

WORKDIR /app

# Copy các file đã build từ giai đoạn builder
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package*.json ./ # Cần cho lệnh start:prod nếu có scripts trong package.json

EXPOSE 3000 # Cổng mà ứng dụng NestJS lắng nghe

CMD ["node", "dist/main"] # Lệnh chạy ứng dụng
Docker Compose: Công cụ để định nghĩa và chạy các ứng dụng đa container Docker. Thường được sử dụng để chạy ứng dụng NestJS cùng với database (ví dụ: PostgreSQL, MongoDB) và Redis trong môi trường phát triển cục bộ.

15.4. CI/CD (GitHub Actions, GitLab CI)
CI/CD (Continuous Integration/Continuous Delivery/Deployment) là một tập hợp các phương pháp và quy trình tự động hóa nhằm cải thiện tốc độ và chất lượng của việc phát triển phần mềm.

Continuous Integration (CI):

Tự động hóa quá trình build và kiểm thử mã nguồn mỗi khi có thay đổi được đẩy lên kho lưu trữ (repository).
Phát hiện lỗi sớm, giảm thiểu rủi ro tích hợp.
Trong NestJS: Chạy npm install, npm run lint, npm run test:e2e, npm run build.
Continuous Delivery (CD):

Đảm bảo mã nguồn đã được kiểm thử và sẵn sàng để triển khai bất cứ lúc nào.
Continuous Deployment (CD):

Tự động triển khai mã nguồn đã được kiểm thử và build thành công lên môi trường sản phẩm.
Các công cụ CI/CD phổ biến:

GitHub Actions: Tích hợp trực tiếp vào GitHub, cho phép bạn tự động hóa các quy trình làm việc (workflows) trực tiếp trong kho lưu trữ của bạn.
Ví dụ Workflow: Khi code được push lên nhánh main, GitHub Actions sẽ tự động cài đặt dependencies, chạy test, build ứng dụng NestJS và sau đó có thể build Docker image và push lên Docker Hub, hoặc triển khai lên dịch vụ đám mây.
GitLab CI/CD: Tích hợp sẵn trong GitLab, cung cấp một hệ thống CI/CD mạnh mẽ thông qua file .gitlab-ci.yml.
Jenkins, CircleCI, Travis CI, Bitbucket Pipelines, AWS CodePipeline, Azure DevOps: Các lựa chọn khác.
15.5. Triển khai lên Heroku, Vercel, VPS
Sau khi ứng dụng NestJS đã được build và đóng gói (nếu dùng Docker), bạn cần một nơi để chạy nó.

Heroku:
Nền tảng PaaS (Platform as a Service) đơn giản, dễ sử dụng.
Hỗ trợ Node.js native, có thể triển khai ứng dụng NestJS bằng cách đẩy code lên Git.
Tự động phát hiện Procfile để chạy lệnh node dist/main.js.
Phù hợp cho các dự án nhỏ và MVP (Minimum Viable Product).
Có các "add-ons" cho database, Redis.
Vercel:
Ban đầu nổi tiếng với các ứng dụng frontend (React, Next.js), nhưng cũng hỗ trợ backend serverless functions (dựa trên Node.js).
Không lý tưởng cho các ứng dụng NestJS monolit (monolithic) chạy liên tục. Tuy nhiên, nếu bạn xây dựng NestJS theo kiến trúc microservices với các Lambda functions, Vercel có thể là một lựa chọn.
Thích hợp hơn cho các API không trạng thái (stateless) và các dự án có yêu cầu backend nhẹ.
VPS (Virtual Private Server):
Cung cấp quyền kiểm soát hoàn toàn hơn so với PaaS. Bạn có một máy chủ ảo riêng để cài đặt hệ điều hành, runtime (Node.js), Docker, Nginx/Caddy (để làm reverse proxy và SSL), và các công cụ quản lý khác.
Các bước triển khai cơ bản:
Thuê VPS (ví dụ: DigitalOcean, Linode, AWS EC2, Google Cloud Compute Engine, Azure VM).
Cài đặt Node.js/Docker lên VPS.
Sao chép code đã build hoặc Docker image lên VPS.
Sử dụng công cụ như PM2 (Process Manager for Node.js) để quản lý tiến trình ứng dụng (đảm bảo ứng dụng luôn chạy, tự khởi động lại khi crash).
Cấu hình reverse proxy (Nginx/Caddy) để xử lý SSL (HTTPS) và định tuyến request đến ứng dụng NestJS.
Lợi ích: Linh hoạt cao, có thể tối ưu hiệu suất theo ý muốn.
Hạn chế: Yêu cầu kiến thức về quản trị hệ thống, cần tự quản lý các vấn đề về bảo mật, scaling.
Việc lựa chọn phương pháp triển khai phụ thuộc vào quy mô dự án, yêu cầu về hiệu suất, ngân sách và mức độ kiểm soát mong muốn.