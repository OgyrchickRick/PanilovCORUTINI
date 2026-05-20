#include <asio.hpp>
#include <iostream>
#include <exception>

using asio::ip::tcp;

asio::awaitable<void> echo_session(tcp::socket sock) {
    char data[1024];
    try {
        for (;;) {
            auto [ec, n] = co_await sock.async_read_some(
                asio::buffer(data),
                asio::as_tuple(asio::use_awaitable)
            );

            if (ec == asio::error::eof) {
                std::cout << "Client disconnected (EOF)" << std::endl;
                break;
            }
            if (ec) {
                throw boost::system::system_error(ec);
            }

            co_await async_write(sock, asio::buffer(data, n), asio::use_awaitable);
        }
    } catch (const std::exception& ex) {
        std::cerr << "Session error: " << ex.what() << std::endl;
    }
}

asio::awaitable<void> echo_server(tcp::acceptor acceptor) {
    try {
        for (;;) {
            tcp::socket socket = co_await acceptor.async_accept(asio::use_awaitable);
            asio::co_spawn(
                acceptor.get_executor(),
                echo_session(std::move(socket)),
                asio::detached
            );
        }
    } catch (const std::exception& ex) {
        std::cerr << "Acceptor error: " << ex.what() << std::endl;
    }
}

int main() {
    try {
        asio::io_context io_context(1);

        tcp::endpoint endpoint(tcp::v4(), 12345);
        tcp::acceptor acceptor(io_context, endpoint);

        std::cout << "Echo server started on port 12345" << std::endl;

        asio::co_spawn(io_context, echo_server(std::move(acceptor)), asio::detached);

        io_context.run();
    } catch (const std::exception& ex) {
        std::cerr << "Main error: " << ex.what() << std::endl;
        return 1;
    }

    return 0;
}