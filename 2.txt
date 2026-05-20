#include <asio.hpp>
#include <asio/experimental/awaitable_operators.hpp>
#include <iostream>
#include <string>
#include <variant>

using asio::ip::tcp;
using namespace asio::experimental::awaitable_operators;

asio::awaitable<std::string> read_from(tcp::socket& sock, const std::string& name) {
    char data[256];
    try {
        auto [ec, n] = co_await sock.async_read_some(
            asio::buffer(data),
            asio::as_tuple(asio::use_awaitable)
        );

        if (ec == asio::error::eof) {
            co_return name + ": DISCONNECTED";
        }
        if (ec) {
            throw boost::system::system_error(ec);
        }

        co_return name + ": " + std::string(data, n);
    } catch (const std::exception& ex) {
        co_return name + ": ERROR - " + ex.what();
    }
}

asio::awaitable<void> multiplexer(tcp::socket sock1, tcp::socket sock2) {
    bool sock1_active = true;
    bool sock2_active = true;

    try {
        while (sock1_active || sock2_active) {
            auto result = co_await (
                (sock1_active ? read_from(sock1, "Sock1") : asio::awaitable<std::string>{} ) ||
                (sock2_active ? read_from(sock2, "Sock2") : asio::awaitable<std::string>{})
            );

            std::visit([&](auto& val) {
                std::cout << val << std::endl;
                if (val.find("DISCONNECTED") != std::string::npos) {
                    if (val.find("Sock1") != std::string::npos) sock1_active = false;
                    if (val.find("Sock2") != std::string::npos) sock2_active = false;
                }
            }, result);
        }
    } catch (const std::exception& ex) {
        std::cerr << "Multiplexer error: " << ex.what() << std::endl;
    }
}

asio::awaitable<tcp::socket> connect_to(const std::string& host, const std::string& port) {
    tcp::resolver resolver(co_await asio::this_coro::executor);
    auto endpoints = co_await resolver.async_resolve(host, port, asio::use_awaitable);
    tcp::socket socket(co_await asio::this_coro::executor);
    co_await asio::async_connect(socket, endpoints, asio::use_awaitable);
    co_return std::move(socket);
}

int main() {
    asio::io_context io_context(1);

    asio::co_spawn(io_context, [&]() -> asio::awaitable<void> {
        tcp::socket sock1(io_context);
        tcp::socket sock2(io_context);
        co_await multiplexer(std::move(sock1), std::move(sock2));
    }, asio::detached);

    io_context.run();
    return 0;
}